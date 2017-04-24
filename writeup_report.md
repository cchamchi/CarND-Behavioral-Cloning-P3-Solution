## Behavioral Cloning Project

The goals / steps of this project are the following:
* Use the simulator to collect data of good driving behavior
* Build, a convolution neural network in Keras that predicts steering angles from images
* Train and validate the model with a training and validation set
* Test that the model successfully drives around track one without leaving the road
* Summarize the results with a written report


[//]: # (Image References)

[recordingerror]: ./writeup_images/.png "Problem recording to directory"
[center]: ./writeup_images/placeholder.png "Image from center camera"
[left]: ./writeup_images/placeholder_small.png "Image from left camera"
[right]: ./writeup_images/placeholder_small.png "Image from right camera"
[centerflipped]: ./writeup_images/placeholder_small.png "Image from center camera, flipped left<->right"
[image6]: ./writeup_images/placeholder_small.png "Normal Image"
[image7]: ./writeup_images/placeholder_small.png "Flipped Image"

## Rubric Points
---
### Files Submitted & Code Quality

#### 1. Submission includes all required files and can be used to run the simulator in autonomous mode

My project includes the following files:
* model.py containing the script to create and train the model
* drive.py for driving the car in autonomous mode
* model.h5 containing a trained convolution neural network 
* writeup_report.md or writeup_report.pdf summarizing the results

#### 2. Submission includes functional code

Once you have cloned this repository, start the Udacity simulator (not provided),
and run
```sh
python drive.py model.h5
```
you should see the car drive around the track autonomously without leaving the road.


#### 3. Submission code is usable and readable

Please refer to model.py.

### Model Architecture and Training Strategy

#### 1. An appropriate model architecture has been employed

My model is a Keras implementation of the Nvidia convolutional neural network designed specifically to generate
steering data for self-driving cars based on camera inputs.  See "Final model architecture" below for a description of the layers.

#### 2. Attempts to reduce overfitting in the model

I split the data into training and validation sets to diagnose overfitting, but when I used the fully augmented data set 
(described in Creation of the Training Set) below, overfitting did not appear to be a significant problem.  Loss on the 
validation set was comparable to loss on the test set at the end of training.  Apparently, the (shuffled and augmented)
training set was large enough to allow the model to generalize to the validation data as well, even without dropout
layers.

I also made sure to monitor the loss while the network was training to make sure the loss was not increasing for later epochs.

####3. Model parameter tuning

I used an Adams optimizer, so tuning learning rate was not necessary.  The one parameter I did tune was the correction
angle added to (subtracted from) the driving angle to pair with an image from the left (right) camera.

After trying several outlier values, I found a range of correction angles that resulted in good driving performance.
I trained the network for correction angles of 0.6, 0.65, 0.7, 0.75 and 0.8.  Training with larger correction angles resulted in 
snappier response to tight turns, but also a tendency to overcorrect on shallower turns, which makes sense.
The model.h5 file accompanying this submission was trained with a correction angle of 0.65.
Sometimes it approaches the side of the road, or sways side to side, but corrects itself robustly.  I actually like the mild
swaying, because it shows the car knows how to recover.

####4. Appropriate training data

See Creation of the Training Set below.

###Model Architecture and Training Strategy

####1. Solution Design Approach

All training was conducted on my laptop, which has TensorFlow equipped 
to use an Nvidia Geforce GTX 960M GPU (Maxwell architecture).

I began by training a 1-layer fully connected network, using only data from the center camera,
just to get the data pipeline working.  

Next I implemented LeNet in Keras, to see how it would perform.  
I trained LeNet using only data from the center camera. 
It sometimes got the car around the first corner and onto the bridge.

I then decided to augment the training dataset by additionally using images from the left and right cameras,
as well as a left-right flipped version of the center camera's image.
This entire training+validation dataset was too large to store in my computer's RAM:
8036 samples x 320x160x3 x 4 bytes per float x 4 images (center,left,right,flipped) = about 20 GB.
It began swapping RAM to the hard drive while running, which made the code infeasibly slow.
I implemented Python generators to serve training and validation data to model.fit_generator().
This made the code run much faster and more smoothly.
However, the car still failed at one of the two sharp curves after the bridge.

I then implemented the Nvidia neural network architecture found here:
[https://devblogs.nvidia.com/parallelforall/deep-learning-self-driving-cars/](https://devblogs.nvidia.com/parallelforall/deep-learning-self-driving-cars/).  
This network was purpose-built for end-to-end training of self-driving car steering based on input from
cameras, so it is ideal for the simulator. 

The only remaining step was to tune the correction applied to the angle associated with the right and left camera images,
as described in "Model parameter tuning" above.  I found that the trained network reliably steered the car all the way around the 
track for several different choices of correction angle.  It was really cool to see how the choice of correction angle influenced
the car's handling.  As I noted earlier, training the network with high correction angles resulted in quick, sharp response to turns, but
also a tendency to overcorrect. Training with smaller correction angles resulted in less swaying back and forth across the road,
but also a gentler (sometimes too gentle) response to sharp turns.

Actually, it was incredibly cool to see the whole thing work, although frustrating at times...but we won't go into that.


####2. Final Model Architecture



![alt text][image1]

####3. Creation of the Training Set & Training Process

Unfortunately, I had trouble recording my own training data on my system (Ubuntu 16.04).  When I tried to select an output
directory from within linux_sim.x86_64, the directory appeared red, and the executable did nothing:

![Recording error][recordingerror]

Choice of directory did not appear to matter.  
The only thing I could think of was that it was a permissions issue, but I chmod 777ed my output
directory, and even ran the simulator as root, and the problem persisted.


I therefore decided to use the provided training data, which was read in from driving_log.csv.
Each line of driving_log.csv corresponded to one sample.
Each sample contained a relative path to center, left, and right camera images, as well as the current driving
angle, throttle, brake, and speed data.

For each data sample, I used all three provided images (from the center, left, and right cameras) 
and also augmented the data with flipped image for the center camera.   

The left (right) camera gives an 
effective view of what the center camera
should see if the car is too far to the left (right); in such cases the car should correct by veering to the 
right (left).  Therefore, to create the angle data for the left camera,
I took the current driving angle and added a correction; to create the angle data for the right camera, I took the driving 
angle and subtracted a correction. 
Using the left and right images this way should aid the car in recovery when the center-camera's image
(which is what is fed to my network in autonomous mode) veers too far to the left or right. 

The angle corresponding to the flipped image was the negative of the current driving angle.
Because the track is counterclockwise, the unaugmented training data contains more left turns than right turns.
Flipping the center-camera image and pairing it with a corresponding flipped angle adds more right-turn data, 
which should help the model generalize.

Images were read in from files, and the flipped image added, using a Python generator.  The generator processed lines of the 
file that stored image locations along with angle data (driving_log.csv) in batches of 32, and supplied 
data to model.fit_generator() in batches of 128 (each line of driving_log.csv was used to provide 
a center-camera, left-camera, right-camera, and center-camera-flipped image). 

The generator also shuffled the array containing the training samples prior to each epoch, so that 
training data would not be fed to the network in the same order.


![center camera][center]
![left camera][left]
![right camera][right]
![center flipped][flipped]

The data set provided 8036 samples, each of which had a center, left, and right image, and I augmented the set 
by flipping the center image.  sklearn.model_selection.train_test_split() was used to split off 20% of the 
data to use for validation.
Therefore, my network was trained on a total of 25,712 data pairs, and validated on a total of 6432 image+angle pairs.

A separate generator was created for training data and validation data. Tthe training generator provided images and angles
derived from samples in the training set, while the validation generator provided images and data derived from samples 
in the validation set).

I trained the model for 5 epochs using an Adams optimizer, which was probably more than necessary, but I wanted to be sure the validation error was plateauing.
I did make sure the validation error was not increasing for later epochs.
