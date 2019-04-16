# Carla Simulator Data Collector
Saving incoming camera sensor images  data as Numpy arrays for generate ground truth data for semantic segmentation.

Task of finding lanes and other obstacles in our path can be greatly simplified by using a neural network capable of semantic segmentation,
because traditional computer vision techniques can’t recognize lane lines, cars, etc. with as much generalization as deep neural networks, 
so we can delegate that task to a semantic segmentation neural network and then build algorithms on top of that. 
So we also want to get semantic segmentation ground truth to train the neural network with.

I referenced the client_example.py file in the PythonClient directory. 
But turns out, the technique used in that script to save the data is awful. They are saving each image (frame) to disk as a .png file 
as it is coming in. This is exactly how not to save data when you want to keep up with a real-time task such as a running simulator, 
because writing to disk is a painfully slow process and waiting for the Python client process to write to disk after each frame causes 
the framerate to drop to about 3-4 fps at best. Hard disks and SSDs alike give the best write speeds 
if you try to write a few large files at once rather than writing many small files. And storing data in RAM is way faster than saving it on disk.
