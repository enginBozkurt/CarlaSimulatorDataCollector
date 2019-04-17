# Carla Simulator Data Collector

<P>
This repo contains all the code for the self-driving project to generate ground truth data for semantic segmentation, 
which in turn makes it much easier to detect not only lanes but also other vehicles and objects in the camera feed, 
and weather and lighting conditions, and a variety of vehicles and roads.
</p>

## Preliminary Setup

You will need Python 3.6 or later to run all the code in this repository.


I would suggest you to get an [Anaconda Python distribution](https://www.anaconda.com/download/) because
it gives you easy access to the entire Scientific Python stack and also makes it really easy to manage
Python environments, even on Windows.

Once you have Anaconda installed, run one of the following from an Anaconda Prompt:

```bash
conda env create -f environment.yml  # without GPU

conda env create -f environment-gpu.yml  # if you have a GPU
```

This will create a conda virtual environment called `sdcar` on your machine. You will then want to use

```bash
activate sdcar
```

to start working in that environment using the installed packages.

Note that if you get the GPU version of Tensorflow from the `environment-gpu.yml` file, you will also need
to follow the instructions on the [TensorFlow website](https://www.tensorflow.org/install/gpu)
in order to allow TensorFlow to use your GPU.

## CARLA Simulator Scripts

The directory `CARLA_simulator_scripts` currently contains the scripts I used to collect camera, depth, and
semantic segmentation ground truth data, but you will need the [CARLA simulator](http://carla.org),
specifically [version 0.8.4](http://carla.org/2018/06/18/release-0.8.4/) in order to run those scripts.

First, you need to download the prebuilt binaries for Windows or Linux from the releases page
[here](https://github.com/carla-simulator/carla/releases/tag/0.8.4) and extract the files in the archive.
In it you will find the simulator and a directory called `PythonClient`. You need to move my scripts into
the `PythonClient` directory. Then open two terminals. In one of the terminals, run the following in order
to start the simulator in server mode (based on your OS):

```bash
# on Linux
./CarlaUE4.sh -carla-server -benchmark -fps=20

# on Windows
CarlaUE4.exe -carla-server -benchmark -fps=20
```

The `-carla-server` flag tells the simulator to run as a server and expect Python clients to connect. The
`-benchmark` flag instructs the simulator to run in
[fixed time-step mode](https://carla.readthedocs.io/en/0.8.4/configuring_the_simulation/#fixed-time-step)
and the `-fps` flag specifies the framerate to run the simulator at. You want to specify a framerate that
your hardware will be able to maintain. This is important because we need to run the simulator in
[synchronous mode](https://carla.readthedocs.io/en/0.8.4/configuring_the_simulation/#synchronous-vs-asynchronous-mode)
while collecting data, otherwise there is a chance that the data captured by the different sensors will
not be synchronized (this will happen if the Python client cannot keep up with the simulation, which is
often the case).

In the other terminal, run the following to start the Python client that will connect to the simulator
server to save the data in the directory `PythonClient/_out`:

```bash
cd PythonClient
python3 manual_control_rgb_semseg.py --images-to-disk --location=_out
```

Everything might take about 20-30 seconds to start up depending on your hardware, but eventually the CARLA
simulator will be launched and a black Pygame window will start. You need to keep the Pygame window in focus
and drive the car manually using the WASD keys. Once you have collected as much data as you want, simply kill
the python process and the simulator. In no time you will find a few gigabytes of Numpy arrays (`.npy` files)
in the `PythonClient/_out` directory. 

You can view the collected data by running the Jupyter Notebook `verify_collected_data.ipynb`. This should also
be run inside the `PythonClient` directory (or you can modify the code inside the notebook to load saved
`.npy` files from whatever location you saved your data to). The notebook is self-explanatory.




## Why I chose to save the data as Numpy arrays instead of images (.png files)

### Getting Data

<p>
The task of finding lanes and other obstacles in our path can be greatly simplified by using a neural network capable of 
semantic segmentation, because traditional computer vision techniques can’t recognize lane lines, cars, etc. 
with as much generalization as deep neural networks, so we can delegate that task to a semantic segmentation neural network 
and then build algorithms on top of that. So we also want to get semantic segmentation ground truth to train the neural network with.
</p>



<p>
Since I wanted to drive the car manually and collect data, I found it easiest to modify the <b>manual_control.py</b> file in the PythonClient directory. 
The final version, <b>manual_control_rgb_semseg.py</b> is in this repository for this project. In order to figure out how to save data, 
I referenced the <b>client_example.py</b> file in the PythonClient directory. But turns out, the technique used in that script to save the data is a little inefficient way.
They are saving each image (frame) to disk as a .png file as it is coming in. This is exactly how not to save data 
when you want to keep up with a real-time task such as a running simulator, because writing to disk is a painfully slow process and 
waiting for the Python client process to write to disk after each frame causes the framerate to drop to about 3-4 fps at best. 
Hard disks and SSDs alike give the best write speeds if you try to write a few large files at once rather than writing many small files. 
And storing data in RAM is way faster than saving it on disk.
</p>


<p>
Finally, since I eventually want to train a neural network with the collected data, it would be really convenient 
if all my collected data were stored in numpy arrays. Then I would not have to open thousands of .png files and read them into memory. 
Storing and retrieving the data in bulk would also be very easy because there would be no need to encode/decode from the PNG format, 
and besides, both opencv and matplotlib work with numpy arrays under the hood, so it does not make visualization any harder.
</p>


## Saving Incoming Data Efficiently
Here is an overview of my idea:

<p>
The data will be stored in a large numpy array as it comes in. This is particularly convenient, 
because the data comes in as 32-bit integers that can be read as 8-bit integers to obtain BGRA images.
<p>

<p>
The data is processed by type.
</p>

<p>
Once the large numpy array (I call it a buffer in the code) fills up, we can write it to disk and make a new, empty buffer. 
The disk write is neither nonblocking nor asynchronous, but it takes less than a second and happens after several minutes of 
uninterrupted gameplay, so overall framerate does not take a hit. 
</p>

<p>
If you take a look at the file <b>buffered_saver.py</b> 
you will find a <b>BufferedImageSaver</b> class which does all the magic. Each <b>BufferedImageSaver</b> object has a buffer (numpy array) 
where it stores the incoming data. Since the numpy array is in memory (RAM), writing to it is very fast. 
Each instance also stores the sensor type associated with it to determine what processing to apply to incoming data. 
The <b>BufferedImageSaver.process_by_type</b> method takes in the raw data provided by the simulator each frame. 
If the sensor is an RGB camera, it does not do anything. But if it is semantic segmentation ground truth, 
then it removes all but the red channel, because it is the only channel with any information.
If the sensor type happens to be a depth camera, it converts the information in the three channels into a single “channel” of 
floating point data.
</p>



<p>
After every frame, the BufferedImageSaver.add_image method is called with the raw sensor data, 
which either stores the data in the buffer, or if the buffer is full, saves the buffer to disk, resets the buffer, 
and then stores the incoming data. 
</p>



Verifying the Saved Data

<p>
I have included a Jupyter Notebook called verify_collected_data.ipynb in the CARLA_simulator_scripts directory which will 
allow you to painlessly visualize the saved data.
</p>

<p>
The visualization process is quite simple: we first load the numpy arrays from disk into memory. 
Now, I lied to you when I said that the camera captures RGB images. It actually saves images in BGR format, 
because Unreal Engine uses the BGRA format for images (it is trivial to get rid of the alpha channel but I did not bother 
to convert from BGR to RGB while saving the numpy arrays in buffered_saver.py because neural networks don’t care either way). 
So we use opencv to convert the images from BGR to RGB in the notebook:
</p>

```python
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
```

<p>
As for the semantic segmentation ground truth arrays, we need to convert the categorical indices (listed here) into actual colors. 
It can be done easily by passing a categorical (qualitative) color map to the cmap argument to the function matplotlib.pyplot.imshow 
as follows:
</p>

```python
plt.imshow(display_array[:, :, 0], cmap='tab20', aspect='auto')
```

<p>
Passing the value 'auto' to the aspect parameter indicates that we want the aspect ratio of the images to be varied to fit 
the given axes. This makes the visualizations better in this case.
</p>

<p>
Below the visualizations is the code I used to generate the images in this blog post. 
Basically, I am converting the categorical semantic segmentation ground truth to RGB using a custom color mapping 
function map_semseg_colors which outputs an RGB image that can then be saved using the pillow (PIL) library. 
</p>
