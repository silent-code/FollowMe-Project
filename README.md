[//]: # (Image References)

[image1]: ./writeup/image1.png
[image3]: ./writeup/image3.png
[image4]: ./writeup/image4.png
[image5]: ./writeup/image5.png
[image6]: ./writeup/image6.png
[image7]: ./writeup/image7.png
[image8]: ./writeup/image8.png

## Project: Follow Me
The Follow Me project goals were to train a fully convolutional neural net (FCN) to allow a simulated quad rotor to track a specified human target while ignoring false human targets. The following images show the performance achieved on test data. The first image below depicts an image of the tracked target (left), the test image ground truth mask (center), and the FCN's performance at labeling the test image (right). The second image shows the FCN's segmentation performance on non-targets. Again, the left image is the test image for non-targets, the center image is the ground truth mask, and the right image shows the FCN's performance at labeling non-targets. As is shown, most of the pixels are labeled correctly in both test images (top and bottom right) and hence the quad rotor will likely track the target well while ignoring false tracks.

![alt text][image6]
![alt text][image7]

### The Fully Convolutional Network 
#### 1. Architecture: One of the primary goals of the project was to design and implemet a fully convolutional neural net. To do so effectively one must construct a model involving image encoding and decoding blocks for image segmentation. 

The fundamental concept is to train the network to label every pixel in an input image. The process of labeling all pixes is called image segmentation. To do effective image segmentation, the network must first encode the image by learning a seriews of filters. Thus the convolutional stages of the network are used to learn image features by increasing the filter space dimensino while simultaneously decreasing the spatial volume dimension. This series of filter-space-increasing / spatial-volume-decreasing layers in often called the encoding block of the FCN as it extracts a filter space encoding of the input training images. Note that separable convolution layers are used as frequently as possible since they perform the filter encoding while decreasing the number of trainable parameters required by reqular convolution layers. Batch normalization is used to provide downstream layers in FCN with inputs that are zero mean and unit variance. A 1x1 convolutional layer is the final encoding stage, providing a means of further expanding the filter dimension while leaving the spatial dimension unchanged. 

The decoding block of layers is used to expand the spatial dimension back to the original input spatial volume while gradually decreasing the filter dimension. Input layers in this block are upsampled and concatenation with the original input image.

The following table shows the network architecture summary (use the command model.summary() before the model.fit_generator call). The layer tensor output shape tuple values are useful for determining spatial volume and filter dimension changes which gives insight into the FCN's information encoding and decoding flow. 


![alt text][image8]


#### 2. Training Parameters: The next step is to specify the network's hyperparameters.

Network hyperparameters for the Follow Me project include the following:

-batch_size: number of training samples/images that get propagated through the network in a single pass.<BR>
-num_epochs: number of times the entire training dataset gets propagated through the network.<BR>
-steps_per_epoch: number of batches of training images that go through the network in 1 epoch. <BR>
-validation_steps: number of batches of validation images that go through the network in 1 epoch. This is similar to steps_per_epoch, except validation_steps is for the validation dataset. <BR>
-workers: maximum number of processes to spin up. This can affect your training speed and is dependent on your hardware.<BR>

Hyperparameters must be specified such that trainable parameters are found for the network that achieve a weighted intersection-over-union (IoU) performance score greater than 0.40. I began hyperparameter optimization by starting with publicly available values used in FCN image segmentation applications. Once I found modifying hyperparameters to be ineffective at final score performance improvements (I was only able to achieve a max final score of 0.39 by tweaking hyperparameters), I turned to modification of network architecture parameters. The breakthrough occured by decreasing the filter dimension reduction between the concatenate_7 and separable_conv2d_keras_25 layers from 70-20 to 70-40. I believe the 70-20 filter dimension change was too extreme and resulted in some of the learned filter encoding information to be lossed in the decoding block. 

The next table shows the hyperparameters I used to achieve a final weighted IoU score of 0.42. 

![alt text][image5]

The final two following images show the the calcuated final weighted IoU score and the 100-epoch training / validation loss curves, respectively.

![alt text][image1]
![alt text][image3]

 
