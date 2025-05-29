1. 🔧 Установка необходимого ПО
Для запуска и компиляции программы вам понадобится:

Visual Studio (рекомендуется версия 2022 или выше)
Бесплатная версия: https://visualstudio.microsoft.com/ru/
Платформа .NET (версия 5.0 или выше)
Скачать можно здесь: https://dotnet.microsoft.com/download
2. 📂 Открытие проекта
Скачайте исходный код или откройте уже имеющийся проект в Visual Studio.
В меню File → Open → Project/Solution , выберите файл CameraApp.csproj или папку с проектом.
3. 📦 Установка библиотек через NuGet
В вашем проекте используются следующие внешние библиотеки, которые нужно установить через NuGet Package Manager :

3.1. Установка OpenCvSharp4
Перейдите в Tools → NuGet Package Manager → Manage NuGet Packages for Solution
Найдите OpenCvSharp4 и OpenCvSharp4.runtime.win
Установите их для вашего проекта.
3.2. Установка NAudio
В том же окне NuGet найдите NAudio
Установите последнюю стабильную версию библиотеки.
⚠️ Примечание: убедитесь, что версии библиотек совместимы с используемой версией .NET. 

4. 🛠 Настройка проекта
4.1. Целевая платформа
В обозревателе решений щёлкните правой кнопкой мыши на вашем проекте → Properties
В разделе Application установите Target framework : .NET 5.0 или .NET 6.0.
4.2. Добавление ссылок
Убедитесь, что подключены следующие сборки:

System.Drawing.Common
System.Windows.Forms
OpenCvSharp
NAudio
Если они не подключены автоматически — добавьте их вручную:

В обозревателе решений → Dependencies → Add Reference
Выберите нужные DLL или добавьте через NuGet, как указано выше.
5. 🎮 Запуск программы
5.1. Компиляция и запуск
Нажмите Build → Build Solution , чтобы убедиться, что проект собран без ошибок.
Нажмите Start (или F5), чтобы запустить приложение.

Исходный код программы управления PTZ-камерой:
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

Form1.designer.cs:

using System.Drawing;
using System.Windows.Forms;

namespace CameraApp
{
    partial class Form1
    {
        private System.ComponentModel.IContainer components = null;
        private PictureBox videoPictureBox;
        private ComboBox cameraComboBox;
        private Button startButton;
        private Button recordButton;
        private TrackBar speedTrackBar;
        private Label speedLabel;
        private Button zoomInButton;
        private Button zoomOutButton;
        private Button panLeftButton;
        private Button panRightButton;
        private Button tiltUpButton;
        private Button tiltDownButton;
        private ComboBox formatComboBox;
        private ComboBox microphoneComboBox;
        private CheckBox recordAudioCheckBox;
        private Button toggleFullScreenButton;

        protected override void Dispose(bool disposing)
        {
            if (disposing && (components != null))
            {
                components.Dispose();
            }
            base.Dispose(disposing);
        }

        private void InitializeComponent()
        {
            this.components = new System.ComponentModel.Container();
            this.videoPictureBox = new PictureBox();
            this.cameraComboBox = new ComboBox();
            this.startButton = new Button();
            this.recordButton = new Button();
            this.speedTrackBar = new TrackBar();
            this.speedLabel = new Label();
            this.zoomInButton = new Button();
            this.zoomOutButton = new Button();
            this.panLeftButton = new Button();
            this.panRightButton = new Button();
            this.tiltUpButton = new Button();
            this.tiltDownButton = new Button();
            this.formatComboBox = new ComboBox();
            this.microphoneComboBox = new ComboBox();
            this.recordAudioCheckBox = new CheckBox();
            this.toggleFullScreenButton = new Button();

            // videoPictureBox
            this.videoPictureBox.Dock = DockStyle.Fill;
            this.videoPictureBox.BackColor = Color.Black;
            this.videoPictureBox.SizeMode = PictureBoxSizeMode.Zoom;
            this.videoPictureBox.Paint += VideoPictureBox_Paint;

            // cameraComboBox
            this.cameraComboBox.Dock = DockStyle.Top;
            this.cameraComboBox.DropDownStyle = ComboBoxStyle.DropDownList;
            this.cameraComboBox.Width = 300;
            this.cameraComboBox.Height = 30;
            this.cameraComboBox.Font = new Font("Arial", 10);
            this.cameraComboBox.Margin = new Padding(0, 5, 0, 10);

            // startButton
            this.startButton.Dock = DockStyle.Top;
            this.startButton.Text = "Запустить";
            this.startButton.FlatStyle = FlatStyle.Flat;
            this.startButton.BackColor = Color.FromArgb(0, 120, 215);
            this.startButton.ForeColor = Color.White;
            this.startButton.Width = 300;
            this.startButton.Height = 40;
            this.startButton.Font = new Font("Arial", 10);
            this.startButton.Click += (s, e) => StartCamera();

            // recordButton
            this.recordButton.Dock = DockStyle.Top;
            this.recordButton.Text = "Начать запись";
            this.recordButton.FlatStyle = FlatStyle.Flat;
            this.recordButton.BackColor = Color.FromArgb(220, 53, 69);
            this.recordButton.ForeColor = Color.White;
            this.recordButton.Width = 300;
            this.recordButton.Height = 40;
            this.recordButton.Font = new Font("Arial", 10);
            this.recordButton.Click += (s, e) => ToggleRecording();

            // speedTrackBar
            this.speedTrackBar.Dock = DockStyle.Top;
            this.speedTrackBar.Minimum = 1;
            this.speedTrackBar.Maximum = 10;
            this.speedTrackBar.Value = 1;
            this.speedTrackBar.TickFrequency = 1;
            this.speedTrackBar.Width = 300;
            this.speedTrackBar.Height = 30;
            this.speedTrackBar.Margin = new Padding(0, 5, 0, 10);
            this.speedTrackBar.Scroll += AdjustSpeed;

            // speedLabel
            this.speedLabel.Dock = DockStyle.Top;
            this.speedLabel.Text = "Скорость: 1";
            this.speedLabel.Font = new Font("Arial", 10);
            this.speedLabel.AutoSize = true;
            this.speedLabel.Margin = new Padding(0, 5, 0, 10);

            // zoomInButton
            this.zoomInButton.Dock = DockStyle.Top;
            this.zoomInButton.Text = "Зум +";
            this.zoomInButton.FlatStyle = FlatStyle.Flat;
            this.zoomInButton.Width = 300;
            this.zoomInButton.Height = 40;
            this.zoomInButton.Font = new Font("Arial", 10);
            this.zoomInButton.Click += (s, e) => ZoomIn();

            // zoomOutButton
            this.zoomOutButton.Dock = DockStyle.Top;
            this.zoomOutButton.Text = "Зум -";
            this.zoomOutButton.FlatStyle = FlatStyle.Flat;
            this.zoomOutButton.Width = 300;
            this.zoomOutButton.Height = 40;
            this.zoomOutButton.Font = new Font("Arial", 10);
            this.zoomOutButton.Click += (s, e) => ZoomOut();

            // panLeftButton
            this.panLeftButton.Dock = DockStyle.Top;
            this.panLeftButton.Text = "◄";
            this.panLeftButton.FlatStyle = FlatStyle.Flat;
            this.panLeftButton.Width = 300;
            this.panLeftButton.Height = 40;
            this.panLeftButton.Font = new Font("Arial", 10);
            this.panLeftButton.Click += (s, e) => PanLeft();

            // panRightButton
            this.panRightButton.Dock = DockStyle.Top;
            this.panRightButton.Text = "►";
            this.panRightButton.FlatStyle = FlatStyle.Flat;
            this.panRightButton.Width = 300;
            this.panRightButton.Height = 40;
            this.panRightButton.Font = new Font("Arial", 10);
            this.panRightButton.Click += (s, e) => PanRight();

            // tiltUpButton
            this.tiltUpButton.Dock = DockStyle.Top;
            this.tiltUpButton.Text = "▲";
            this.tiltUpButton.FlatStyle = FlatStyle.Flat;
            this.tiltUpButton.Width = 300;
            this.tiltUpButton.Height = 40;
            this.tiltUpButton.Font = new Font("Arial", 10);
            this.tiltUpButton.Click += (s, e) => TiltUp();

            // tiltDownButton
            this.tiltDownButton.Dock = DockStyle.Top;
            this.tiltDownButton.Text = "▼";
            this.tiltDownButton.FlatStyle = FlatStyle.Flat;
            this.tiltDownButton.Width = 300;
            this.tiltDownButton.Height = 40;
            this.tiltDownButton.Font = new Font("Arial", 10);
            this.tiltDownButton.Click += (s, e) => TiltDown();

            // formatComboBox
            this.formatComboBox.Dock = DockStyle.Top;
            this.formatComboBox.DropDownStyle = ComboBoxStyle.DropDownList;
            this.formatComboBox.Items.AddRange(new string[] { "MP4", "AVI", "MKV" });
            this.formatComboBox.SelectedIndex = 0;
            this.formatComboBox.Width = 300;
            this.formatComboBox.Height = 30;
            this.formatComboBox.Font = new Font("Arial", 10);
            this.formatComboBox.Margin = new Padding(0, 5, 0, 10);

            // microphoneComboBox
            this.microphoneComboBox.Dock = DockStyle.Top;
            this.microphoneComboBox.DropDownStyle = ComboBoxStyle.DropDownList;
            this.microphoneComboBox.Width = 300;
            this.microphoneComboBox.Height = 30;
            this.microphoneComboBox.Font = new Font("Arial", 10);
            this.microphoneComboBox.Margin = new Padding(0, 5, 0, 10);

            // recordAudioCheckBox
            this.recordAudioCheckBox.Dock = DockStyle.Top;
            this.recordAudioCheckBox.Text = "Записывать звук";
            this.recordAudioCheckBox.Checked = false;
            this.recordAudioCheckBox.Font = new Font("Arial", 10);
            this.recordAudioCheckBox.AutoSize = true;
            this.recordAudioCheckBox.Margin = new Padding(0, 5, 0, 10);

            // toggleFullScreenButton
            this.toggleFullScreenButton.Dock = DockStyle.Top;
            this.toggleFullScreenButton.Text = "Оконный режим";
            this.toggleFullScreenButton.FlatStyle = FlatStyle.Flat;
            this.toggleFullScreenButton.BackColor = Color.FromArgb(108, 117, 125);
            this.toggleFullScreenButton.ForeColor = Color.White;
            this.toggleFullScreenButton.Width = 300;
            this.toggleFullScreenButton.Height = 40;
            this.toggleFullScreenButton.Font = new Font("Arial", 10);
            this.toggleFullScreenButton.Margin = new Padding(0, 15, 0, 15);
            this.toggleFullScreenButton.Click += (s, e) =>
            {
                this.WindowState = this.WindowState == FormWindowState.Maximized ? FormWindowState.Normal : FormWindowState.Maximized;
                this.toggleFullScreenButton.Text = this.WindowState == FormWindowState.Maximized ? "Оконный режим" : "Полноэкранный режим";
            };

            // MainForm
            this.Text = "Камера: Управление и Настройки";
            this.WindowState = FormWindowState.Maximized;
            this.MinimumSize = new System.Drawing.Size(640, 480);
            this.FormClosing += Form1_FormClosing;

            // Panels
            var mainPanel = new Panel { Dock = DockStyle.Fill };
            var controlPanel = new Panel { Dock = DockStyle.Right, Width = 350, BackColor = Color.FromArgb(240, 240, 240) };

            // Add videoPictureBox to mainPanel
            mainPanel.Controls.Add(videoPictureBox);

            // Add controls to controlPanel in order
            controlPanel.Controls.AddRange(new Control[]
            {
                cameraComboBox,
                startButton,
                recordButton,
                speedLabel,
                speedTrackBar,
                zoomInButton,
                zoomOutButton,
                panLeftButton,
                panRightButton,
                tiltUpButton,
                tiltDownButton,
                formatComboBox,
                microphoneComboBox,
                recordAudioCheckBox,
                toggleFullScreenButton
            });

            // Add panels to form
            this.Controls.Add(mainPanel);
            this.Controls.Add(controlPanel);
        }
    }
}