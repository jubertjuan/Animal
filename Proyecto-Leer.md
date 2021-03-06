# Animal-Finalizado 
Finalizar Proyecto


Asi quedo todo el Proyecto Finalizado

using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

using AForge;
using AForge.Imaging;
using AForge.Imaging.Filters;
using AForge.Video;
using AForge.Video.DirectShow;

using System.Drawing.Imaging;

namespace Color_Detection_Using_AForge.NET
{
    public partial class MainForm : Form
    {
        #region Global Variables
        // List of video devices
        private FilterInfoCollection videoDevices;
        private VideoCaptureDevice videoSource;

        // Form to tune object detection filter
        private TuneObjectFilterForm tuneObjectFilterForm;

        private ColorFiltering colorFilter = new ColorFiltering();
        private GrayscaleBT709   grayFilter = new GrayscaleBT709();

        private BlobCounter blobCounter = new BlobCounter();

        #endregion Global Variables

        #region Constructor
        public MainForm()
        {
            InitializeComponent();

            colorFilter.Red = new IntRange(0,255);    
            colorFilter.Green = new IntRange(0,255);   
            colorFilter.Blue = new IntRange(0,255); 
            

            // configure blob counters
            blobCounter.MinWidth = 15;
            blobCounter.MinHeight = 15;
            blobCounter.FilterBlobs = true;
            blobCounter.ObjectsOrder = ObjectsOrder.Size;

            EnableConnectionControls(true);
        }

        #endregion Constructor

        #region Methods
        private void EnableConnectionControls(bool enable)
        {
            cmbVideoDevices.Enabled = enable;
            btnRefreshCameraDevices.Enabled = enable;
            gbObjectDetection.Enabled = !enable;
        }

        private void ConnectCamera() 
        {
            if (videoSource != null)
            {
                videoSourcePlayer.VideoSource = videoSource;
                videoSourcePlayer.Start();
            }
        }

        private void DisconnectCamera()
        {
            if (videoSourcePlayer.VideoSource != null)
            {
                // Stop video device
                videoSourcePlayer.SignalToStop();
                videoSourcePlayer.WaitForStop();
                videoSourcePlayer.VideoSource = null;
            }
        }

        private void RefreshVideoDevices()
        {
            cmbVideoDevices.Items.Clear();

            // Enumerate video devices
            videoDevices = new FilterInfoCollection(FilterCategory.VideoInputDevice);

            if (videoDevices.Count != 0)
            {
                // Add all devices to combobox
                foreach (FilterInfo device in videoDevices)
                {
                    cmbVideoDevices.Items.Add(device.Name);
                }
            }
            else
            {
                cmbVideoDevices.Items.Add("No hay dispositivos DirectShow encontrados");
            }

            cmbVideoDevices.SelectedIndex = 0;
        }

        private void ProcessImage(Bitmap image)
        {
            Bitmap objectImage = colorFilter.Apply(image);
            // Lock image
            BitmapData objectData = objectImage.LockBits(new Rectangle(0, 0, image.Width, image.Height),
                                                            ImageLockMode.ReadOnly, image.PixelFormat);
            // Grayscaling
            UnmanagedImage grayImage = grayFilter.Apply(new UnmanagedImage(objectData));                 
                  
            
            objectImage.UnlockBits(objectData);
            blobCounter.ProcessImage(grayImage);

            Rectangle[] rects = blobCounter.GetObjectsRectangles();

            for  (int i=0, n=rects.Length; i < n; i++ )
            {
                
                Rectangle objectRect = rects[i];
                // Draw rectangle around derected object
                Graphics g = Graphics.FromImage(image);

                //System.Drawing.Point centerOfMass = new System.Drawing.Point(objectRect.X + objectRect.Width / 2,
                  //                                      objectRect.Y + objectRect.Height / 2);

               // using (Pen pen = new Pen(Color.FromArgb(255, 128, 0), 5))   // Orange color pen
                //using (Pen pen = new Pen(Color.FromArgb(int.Parse(tuneObjectFilterForm.RedRange.Length.ToString()),
                //int.Parse(tuneObjectFilterForm.GreenRange.Length.ToString()),
                //int.Parse(tuneObjectFilterForm.BlueRange.Length.ToString())), 5))
                using (Pen pen = new Pen(Color.Red, 5))
                {
                    g.DrawRectangle(pen, objectRect);
                    //g.DrawEllipse(pen, centerOfMass.X, centerOfMass.Y, 5, 5);
                }
                g.Dispose();
            }
        }

        #endregion Methods

        #region Events
        private void MainForm_Load(object sender, EventArgs e)
        {
            RefreshVideoDevices();
        }

        private void MainForm_FormClosing(object sender, FormClosingEventArgs e)
        {
            DisconnectCamera();
        }

        private void btnStartStopCamera_Click(object sender, EventArgs e)
        {
            if (btnStartStopCamera.Text == "Iniciar")
            {
                ConnectCamera();
                btnStartStopCamera.Text = "Detener";
                                


                EnableConnectionControls(false);
            }
            else
            {
                DisconnectCamera();
                btnStartStopCamera.Text = "Iniciar";
                EnableConnectionControls(true);
            }
        }

        private void cmbVideoDevices_SelectedIndexChanged(object sender, EventArgs e)
        {
            if (videoDevices.Count != 0)
            {
                videoSource = new VideoCaptureDevice(videoDevices[cmbVideoDevices.SelectedIndex].MonikerString);
            }
        }

        private void btnRefreshCameraDevices_Click(object sender, EventArgs e)
        {
            RefreshVideoDevices();
        }

        private void btnExit_Click(object sender, EventArgs e)
        {
            Application.Exit();
            
        }

        private void btnTuneObjectFilter_Click(object sender, EventArgs e)
        {
            if (tuneObjectFilterForm == null)
            {
                tuneObjectFilterForm = new TuneObjectFilterForm();
                tuneObjectFilterForm.OnFilterUpdate += new EventHandler(tuneObjectFilterForm_OnFilterUpdate);

                tuneObjectFilterForm.RedRange = colorFilter.Red;
                tuneObjectFilterForm.GreenRange = colorFilter.Green;
                tuneObjectFilterForm.BlueRange = colorFilter.Blue;
            }
            tuneObjectFilterForm.Show();
        }

        private void tuneObjectFilterForm_OnFilterUpdate(object sender, EventArgs e)
        {
            colorFilter.Red = tuneObjectFilterForm.RedRange;
            colorFilter.Green = tuneObjectFilterForm.GreenRange;
            colorFilter.Blue = tuneObjectFilterForm.BlueRange;
        }

        private void videoSourcePlayer_NewFrame(object sender, ref Bitmap image)
        {
            bool showOnlyObjects = chkShowDetectedObjects.Checked;

            Bitmap objectImage = null;

            // filtrado de color
            if (showOnlyObjects)
            {
                objectImage = image;
                colorFilter.ApplyInPlace(image);
            }
            else
            {
                objectImage = colorFilter.Apply(image);
            }

            if (chkObjectDetection.Checked)
            {
                ProcessImage(image);
            }
        }


        #endregion Events

        private void btnRojo_Click(object sender, EventArgs e)
        {
            colorFilter.Red = new IntRange(0, 255);
            colorFilter.Green = new IntRange(0, 0);
            colorFilter.Blue = new IntRange(0, 0);
        }

        private void btnVerde_Click(object sender, EventArgs e)
        {
            colorFilter.Red = new IntRange(0,0);
            colorFilter.Green = new IntRange(0,255);
            colorFilter.Blue = new IntRange(0,0);
        }

        private void btnAzul_Click(object sender, EventArgs e)
        {
            colorFilter.Red = new IntRange(0,0);
            colorFilter.Green = new IntRange(0,0);
            colorFilter.Blue = new IntRange(0, 255);
        }

        private void btnSave_Click(object sender, EventArgs e)
        {
            //DIALOGO PARA SELECCIONAR LA RUTA PARA GUARDAR
            SaveFileDialog sf = new SaveFileDialog();
            //FILTO DE IMAGENES JPG
            sf.Filter = "Imagenes JPG | *.jpg";
            //MOSTRAR DIALOGO
            sf.ShowDialog();
            //ASEGURAR QUE TIENE UNA RUTA VALIDA
            if (sf.FileName != null)
            {
                //VARIBALE PARA LA IMAGEN
                Bitmap img = videoSourcePlayer.GetCurrentVideoFrame();
                //GUARDAR IMAGEN EN LA RUTA
                img.Save(sf.FileName, System.Drawing.Imaging.ImageFormat.Jpeg);
                //BORRAR IMAGEN DE MEMORIA
                img.Dispose();
                //PROBAR!
            }
        }

      
    } // class
} // namespace

