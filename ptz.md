Form1.cs:

using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Windows.Forms;
using OpenCvSharp;
using OpenCvSharp.Extensions;
using NAudio.Wave;

namespace CameraApp
{
    public partial class Form1 : Form
    {
        private VideoCapture capture;
        private bool isRecording = false;
        private VideoWriter videoWriter;
        private WaveInEvent waveIn;
        private WaveFileWriter waveFileWriter;
        private string finalVideoFilePath;
        private string finalAudioFilePath;
        private int speedMultiplier = 1;
        private double zoomFactor = 1.0;
        private bool recordAudio = false;
        private Timer recordingIndicatorTimer;
        private bool recordingIndicatorVisible = false;
        public Form1()
        {
            InitializeComponent();
            FindAvailableCameras();
            PopulateMicrophones();
            this.Resize += Form1_Resize;
            videoPictureBox.MinimumSize = new System.Drawing.Size(1, 1);
            SetupRecordingIndicator();
        }
        private void SetupRecordingIndicator()
        {
            recordingIndicatorTimer = new Timer
            {
                Interval = 500
            };
            recordingIndicatorTimer.Tick += (s, e) =>
            {
                recordingIndicatorVisible = !recordingIndicatorVisible;
                videoPictureBox.Invalidate();
            };
        }
        private void FindAvailableCameras()
        {
            for (int i = 0; i < 10; i++)
            {
                using (var cap = new VideoCapture(i))
                {
                    if (cap.IsOpened())
                    {
                        cameraComboBox.Items.Add($"Камера {i}");
                        cap.Release();
                    }
                }
            }
            if (cameraComboBox.Items.Count > 0)
                cameraComboBox.SelectedIndex = 0;
            else
                MessageBox.Show("Камеры не найдены.", "Предупреждение", MessageBoxButtons.OK, MessageBoxIcon.Warning);
        }
        private void PopulateMicrophones()
        {
            microphoneComboBox.Items.Clear();
            int deviceCount = WaveIn.DeviceCount;
            if (deviceCount == 0)
            {
                MessageBox.Show("Микрофоны не найдены. Запись звука будет отключена.", "Предупреждение", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                recordAudioCheckBox.Enabled = false;
                return;
            }
            for (int i = 0; i < deviceCount; i++)
            {
                var capabilities = WaveIn.GetCapabilities(i);
                microphoneComboBox.Items.Add(capabilities.ProductName);
            }
            if (microphoneComboBox.Items.Count > 0)
                microphoneComboBox.SelectedIndex = 0;
        }
        private void StartCamera()
        {
            if (cameraComboBox.SelectedIndex == -1)
            {
                MessageBox.Show("Камера не выбрана.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }
            int selectedCamera = cameraComboBox.SelectedIndex;
            capture = new VideoCapture(selectedCamera);
            if (!capture.IsOpened())
            {
                MessageBox.Show("Не удалось открыть камеру.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }
            using (var testFrame = capture.RetrieveMat())
            {
                if (testFrame.Empty())
                {
                    MessageBox.Show("Камера не возвращает изображение.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    capture.Release();
                    return;
                }
            }
            Application.Idle += UpdateVideo;
        }
        private void UpdateVideo(object sender, EventArgs e)
        {
            using (var frame = capture.RetrieveMat())
            {
                if (frame.Empty() || frame.Width <= 0 || frame.Height <= 0)
                {
                    return;
                }
                if (isRecording && videoWriter != null)
                {
                    if (frame.Width == videoWriter.FrameSize.Width && frame.Height == videoWriter.FrameSize.Height)
                    {
                        videoWriter.Write(frame);
                    }
                    else
                    {
                        MessageBox.Show("Размеры кадра не совпадают с параметрами записи.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        isRecording = false;
                        recordButton.Text = "Начать запись";
                        videoWriter?.Release();
                        videoWriter = null;
                    }
                }
                var zoomedFrame = ApplyZoom(frame);
                using (var resizedFrame = ResizeFrameToFit(zoomedFrame, new System.Drawing.Size(videoPictureBox.Width, videoPictureBox.Height)))
                using (var bitmap = BitmapConverter.ToBitmap(resizedFrame))
                {
                    videoPictureBox.Image?.Dispose();
                    videoPictureBox.Image = new Bitmap(bitmap);
                }
            }
        }
        private bool SupportsHardwareZoom()
        {
            if (capture == null || !capture.IsOpened())
                return false;
            double zoomValue = capture.Get(VideoCaptureProperties.Zoom);
            return !double.IsNaN(zoomValue) && zoomValue >= 0;
        }
        private void ApplyHardwareZoom(double zoom)
        {
            if (capture != null && capture.IsOpened())
            {
                capture.Set(VideoCaptureProperties.Zoom, zoom);
            }
        }
        private Mat ApplyZoom(Mat frame)
        {
            if (Math.Abs(zoomFactor - 1.0) < 0.01 || SupportsHardwareZoom())
            {
                return frame;
            }
            var zoomedFrame = new Mat();
            Cv2.Resize(frame, zoomedFrame, new OpenCvSharp.Size((int)(frame.Width * zoomFactor), (int)(frame.Height * zoomFactor)));
            return zoomedFrame;
        }
        private Mat ResizeFrameToFit(Mat frame, System.Drawing.Size pictureBoxSize)
        {
            if (pictureBoxSize.Width <= 0 || pictureBoxSize.Height <= 0)
            {
                return frame;
            }
            var openCvSize = new OpenCvSharp.Size(pictureBoxSize.Width, pictureBoxSize.Height);
            double ratio = Math.Min(openCvSize.Width / frame.Width, openCvSize.Height / frame.Height);
            if (ratio <= 0)
            {
                return frame;
            }
            int newWidth = (int)(frame.Width * ratio);
            int newHeight = (int)(frame.Height * ratio);
            var resizedFrame = new Mat();
            Cv2.Resize(frame, resizedFrame, new OpenCvSharp.Size(newWidth, newHeight));
            return resizedFrame;
        }
        private void ToggleRecording()
        {
            if (capture == null || !capture.IsOpened())
            {
                MessageBox.Show("Камера не запущена.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }
            if (!isRecording)
            {
                // Video file selection
                var videoSaveFileDialog = new SaveFileDialog
                {
                    Title = "Сохранить видео как"
                };
                string selectedFormat = formatComboBox.SelectedItem.ToString();
                switch (selectedFormat)
                {
                    case "MP4":
                        videoSaveFileDialog.Filter = "MP4 Files (*.mp4)|*.mp4";
                        videoSaveFileDialog.DefaultExt = ".mp4";
                        break;
                    case "AVI":
                        videoSaveFileDialog.Filter = "AVI Files (*.avi)|*.avi";
                        videoSaveFileDialog.DefaultExt = ".avi";
                        break;
                    case "MKV":
                        videoSaveFileDialog.Filter = "MKV Files (*.mkv)|*.mkv";
                        videoSaveFileDialog.DefaultExt = ".mkv";
                        break;
                    default:
                        MessageBox.Show("Неподдерживаемый формат.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        return;
                }
                if (videoSaveFileDialog.ShowDialog() != DialogResult.OK)
                    return;
                finalVideoFilePath = videoSaveFileDialog.FileName;
                // Audio file selection (if enabled)
                finalAudioFilePath = null;
                recordAudio = recordAudioCheckBox.Checked && recordAudioCheckBox.Enabled;
                if (recordAudio)
                {
                    var audioSaveFileDialog = new SaveFileDialog
                    {
                        Title = "Сохранить аудио как",
                        Filter = "WAV Files (*.wav)|*.wav",
                        DefaultExt = ".wav",
                        FileName = Path.GetFileNameWithoutExtension(finalVideoFilePath) + "_audio"
                    };
                    if (audioSaveFileDialog.ShowDialog() != DialogResult.OK)
                    {
                        recordAudio = false;
                        recordAudioCheckBox.Checked = false;
                    }
                    else
                    {
                        finalAudioFilePath = audioSaveFileDialog.FileName;
                    }
                }
                int width = GetValidValue((int)capture.Get(VideoCaptureProperties.FrameWidth), 640);
                int height = GetValidValue((int)capture.Get(VideoCaptureProperties.FrameHeight), 480);
                double fps = GetValidValue(capture.Get(VideoCaptureProperties.Fps), 30);
                if (width <= 0 || height <= 0 || fps <= 0)
                {
                    MessageBox.Show("Некорректные параметры видеопотока.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    return;
                }
                FourCC fourcc;
                switch (selectedFormat)
                {
                    case "MP4":
                        fourcc = FourCC.MP4V;
                        break;
                    case "AVI":
                        fourcc = FourCC.XVID;
                        break;
                    case "MKV":
                        MessageBox.Show("MKV не поддерживает специфичные кодеки. Будет использован MJPG.", "Предупреждение", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                        fourcc = FourCC.MJPG;
                        break;
                    default:
                        MessageBox.Show("Неподдерживаемый формат.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        return;
                }
                videoWriter = new VideoWriter(finalVideoFilePath, fourcc, fps, new OpenCvSharp.Size(width, height));
                if (!videoWriter.IsOpened())
                {
                    MessageBox.Show("Не удалось начать запись.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    return;
                }
                if (recordAudio)
                {
                    StartAudioRecording();
                }
                isRecording = true;
                recordButton.Text = "Остановить запись";
                recordingIndicatorTimer.Start();
            }
            else
            {
                isRecording = false;
                recordButton.Text = "Начать запись";
                recordingIndicatorTimer.Stop();
                recordingIndicatorVisible = false;
                videoPictureBox.Invalidate();
                videoWriter?.Release();
                videoWriter = null;
                if (recordAudio)
                {
                    StopAudioRecording();
                }
                if (recordAudio && finalAudioFilePath != null)
                {
                    MessageBox.Show($"Видео сохранено в: {finalVideoFilePath}\nАудио сохранено в: {finalAudioFilePath}", "Запись завершена", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
                else
                {
                    MessageBox.Show($"Видео сохранено в: {finalVideoFilePath}", "Запись завершена", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
            }
        }
        private void StartAudioRecording()
        {
            try
            {
                if (microphoneComboBox.SelectedIndex == -1)
                {
                    MessageBox.Show("Микрофон не выбран.", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    recordAudio = false;
                    recordAudioCheckBox.Checked = false;
                    return;
                }
                if (File.Exists(finalAudioFilePath))
                    File.Delete(finalAudioFilePath);
                waveIn = new WaveInEvent
                {
                    DeviceNumber = microphoneComboBox.SelectedIndex,
                    WaveFormat = new WaveFormat(44100, 16, 1)
                };
                waveFileWriter = new WaveFileWriter(finalAudioFilePath, waveIn.WaveFormat);
                waveIn.DataAvailable += OnAudioDataAvailable;
                waveIn.RecordingStopped += (s, e) =>
                {
                    waveIn?.Dispose();
                    waveIn = null;
                    waveFileWriter?.Dispose();
                    waveFileWriter = null;
                };
                waveIn.StartRecording();
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Ошибка при запуске записи звука: {ex.Message}", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                recordAudio = false;
                recordAudioCheckBox.Checked = false;
                waveIn?.Dispose();
                waveFileWriter?.Dispose();
            }
        }
        private void StopAudioRecording()
        {
            waveIn?.StopRecording();
            waveIn?.Dispose();
            waveIn = null;
            waveFileWriter?.Dispose();
            waveFileWriter = null;
        }
        private void OnAudioDataAvailable(object sender, WaveInEventArgs e)
        {
            if (waveFileWriter != null)
            {
                waveFileWriter.Write(e.Buffer, 0, e.BytesRecorded);
                waveFileWriter.Flush();
            }
        }
        private int GetValidValue(int value, int defaultValue)
        {
            return value > 0 ? value : defaultValue;
        }
        private double GetValidValue(double value, double defaultValue)
        {
            return value > 0 ? value : defaultValue;
        }
        private void AdjustSpeed(object sender, EventArgs e)
        {
            speedMultiplier = speedTrackBar.Value;
            speedLabel.Text = $"Скорость: {speedMultiplier}";
        }
        private void ZoomIn()
        {
            zoomFactor += 0.1;
            if (SupportsHardwareZoom())
            {
                ApplyHardwareZoom(zoomFactor * 100);
            }
        }
        private void ZoomOut()
        {
            zoomFactor = Math.Max(zoomFactor - 0.1, 0.1);
            if (SupportsHardwareZoom())
            {
                ApplyHardwareZoom(zoomFactor * 100);
            }
        }
        private void PanLeft()
        {
            if (capture != null)
            {
                double currentPan = capture.Get(VideoCaptureProperties.Pan);
                capture.Set(VideoCaptureProperties.Pan, currentPan - speedMultiplier);
            }
        }
        private void PanRight()
        {
            if (capture != null)
            {
                double currentPan = capture.Get(VideoCaptureProperties.Pan);
                capture.Set(VideoCaptureProperties.Pan, currentPan + speedMultiplier);
            }
        }
        private void TiltUp()
        {
            if (capture != null)
            {
                double currentTilt = capture.Get(VideoCaptureProperties.Tilt);
                capture.Set(VideoCaptureProperties.Tilt, currentTilt + speedMultiplier);
            }
        }
        private void TiltDown()
        {
            if (capture != null)
            {
                double currentTilt = capture.Get(VideoCaptureProperties.Tilt);
                capture.Set(VideoCaptureProperties.Tilt, currentTilt - speedMultiplier);
            }
        }
        private void Form1_FormClosing(object sender, FormClosingEventArgs e)
        {
            if (capture != null && capture.IsOpened())
                capture.Release();
            if (waveIn != null)
                waveIn.Dispose();
            if (waveFileWriter != null)
                waveFileWriter.Dispose();
            if (videoWriter != null)
                videoWriter.Release();
            recordingIndicatorTimer?.Dispose();
        }
        private void Form1_Resize(object sender, EventArgs e)
        {
            if (videoPictureBox != null)
            {
                videoPictureBox.Invalidate();
            }
        }
        private void VideoPictureBox_Paint(object sender, PaintEventArgs e)
        {
            if (isRecording && recordingIndicatorVisible)
            {
                using (var brush = new SolidBrush(Color.Red))
                {
                    e.Graphics.FillEllipse(brush, 10, 10, 20, 20);
                }
                using (var font = new Font("Arial", 12))
                using (var brush = new SolidBrush(Color.Red))
                {
                    e.Graphics.DrawString("Recording", font, brush, 35, 10);
                }
            }
        }
    }
}
