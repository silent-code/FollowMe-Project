[//]: # (Image References)

[image1]: ./image1.png
[image3]: ./image3.png
[image4]: ./image4.png
[image5]: ./image5.png
[image6]: ./image6.png
[image7]: ./image7.png
[image8]: ./image8.png

## Project: Follow Me
The Follow Me project goals were to train a fully convolutional neural net (FCN) to allow a simulated quad rotor to track a specified human target while ignoring false human targets. The following images show the performance achieved on test data. The first image below depicts an image of the tracked target (left), the test image ground truth mask (center), and the FCN's performance at labeling the test image (right). The second image shows the FCN's segmentation performance on non-targets. Again, the left image is the test image for non-targets, the center image is the ground truth mask, and the right image shows the FCN's performance at labeling non-targets. As is shown, most of the pixels are labeled correctly in both test images (top and bottom right) and hence the quad rotor will likely track the target well while ignoring false tracks.

![alt text][image6]
![alt text][image7]

### The Fully Convolutional Network 
#### 1. Architecture: One of the primary goals of the project was to design and implemet a fully convolutional neural net. To do so effectively one must construct a model involving image encoding and decoding blocks for image segmentation. 

The fundamental concept is to train the network to label every pixel in an input image. The process labeling all pixes is called image segmentation. To do effective image segmentation, the network must first encode the image by learning a seriews of filters. Thus the convolutional stages of the network are used to learn image features by increasing the filter space while simultaneously decreasing the spatial volume. This series of filter-space-increasing / spatial-volume-decreasing layers in often called the encoding block of the FCN as it extracts a filter space encoding of the input training images. Note that separable convolution layers are used as frequently since they perform the encoding while decreasing the number of trainable parameters required by reqular convolution layers. Batch normalization is used to provide downstream layers in FCN with inputs that are zero mean and unit variance. A 1x1 convolutional layer is the final encoding stage, providing a means of further expanding the filter dimension while leaving the spatial dimension unchanged. 

The decoding block of layers is used to expand the spatial dimension back to the original input spatial volume while gradually decreasing the filter dimension. Input layers in this block are upsampled and concatenation with the original input image.

The following tables shows the network architecture summary. The layer tensor output shape is useful for determining spatial volume and filter dimension changes which gives insight into the FCN's information encoding and decoding flow. 


![alt text][image8]


#### 2. Training Parameters: The next step is to specify the network's hyperparameters.

Network hyperparameters for the Follow Me project include
-batch_size: number of training samples/images that get propagated through the network in a single pass.
-num_epochs: number of times the entire training dataset gets propagated through the network.
-steps_per_epoch: number of batches of training images that go through the network in 1 epoch. We have provided you with a default value. One recommended value to try would be based on the total number of images in training dataset divided by the batch_size.
-validation_steps: number of batches of validation images that go through the network in 1 epoch. This is similar to steps_per_epoch, except validation_steps is for the validation dataset. We have provided you with a default value for this as well.
-workers: maximum number of processes to spin up. This can affect your training speed and is dependent on your hardware. We have provided a recommended value to work with.

Hyperparameters must be specified such that trainable parameters are found for the network that allow for a weighted intersection over union performance score greater than 0.40 is achieved. The next table shows the hyperparameters I used to achieve a final score of 0.42. 

![alt text][image5]

The next image shows the 100-epoch training and validation loss curves.

![alt text][image3]


 
#### 3. Next we were required to decouple the Inverse Kinematics problem into Inverse Position Kinematics and Inverse Orientation Kinematics and by doing so obtain the equations to calculate all individual joint angles.

First we needed to obtain the location of the wrist center position [WCx, WCy, WCz]. Since the Kuka has a spherical wrist, the wrist position and orientation with respect to the robot base are independent. The following diagram shows the wrist center position derivation.  

![alt text][image4]

Once we have the wrist center position it is fairly straight forward to derive the first three joint angles. The next diagram shows the pertinent robot geometry between the base and wrist center (WC). 

![alt text][image3]

To calculate the final three joint angles we use the fact that the Grip rotation matrix with respect to the base, Rot_ee, is equal to the matrix composition R0_3 * R3_6. Thus

R3_6 = R0_3.inv("LU") * Rot_ee

The above right hand side is a numerical 3 x 3 matrix since the first three joint angles are known (i.e., we can numerically compute R0_3) and we can compute Rot_ee given the robot Grip orientation. Since the left hand side contains the final three joint variables we can use these equations to solve for them numericallhy. One can use the IK_debug.py with the following code added to print out and solve for the joint variables 4-6. For example equating the [1, 1]-element: 

T3_6 = T3_4 * T4_5 * T5_6<br>
R3_6 = T3_6[0:3, 0:3]<br>
R3_6_sym = simplify(R3_6.T * Rot_ee_symbol)<br>
R3_6_sym = R3_6_sym.subs({'r': roll, 'p': pitch, 'y': yaw})<br>
print(np.matrix(R3_6_sym[1,1]))<br>
R0_3 = R0_3.evalf(subs={q1: theta1, q2: theta2, q3: theta3})<br>
R3_6 = R0_3.inv("LU") * Rot_ee<br>
print(np.matrix(R3_6[1,1]))<br>

Yields the equation:

-0.332 = 0.879 * sin(theta4) * sin(theta5 - theta6) - 0.473 * sin(theta5 - theta6) * cos(theta4) - 0.054 * cos(theta5 - theta6)

The final equations in python are:

theta4 = atan2(R3_6[2, 2], -R3_6[0, 2])<br>
theta5 = atan2(sqrt(R3_6[0, 2] * R3_6[0, 2] + R3_6[2, 2] * R3_6[2, 2]), R3_6[1, 2])<br>
theta6 = atan2(-R3_6[1, 1], R3_6[1, 0])<br>

### Project Implementation

#### 1. To complete the project I filled in the `IK_server.py` file with the forward and inverse kinematic code we implemented in the IK_debug.py python code for calculating Inverse Kinematics based on previously performed Kinematic Analysis. The code has been shown to guide the robot to successfully complete at least 8/10 pick and place cycles. The final image is a screen cap showing the successful placement of the 8 objects in the bin and the two failed attempts on the shelf. The final standing object on the shelf is simply the 11th attempt at reset which wasn't used.   




