[//]: # (Image References)

[image1]: ./image1.png
[image3]: ./image3.png
[image4]: ./image4.png
[image5]: ./image5.png
[image6]: ./image6.png
[image7]: ./image7.png
[image8]: ./Screen Shot 2019-01-03 at 1.47.22 PM.png

## Project: Follow Me
The Follow Me project goals were to train a fully convolutional neural net (FCN) to allow a simulated quad rotor to track a specified human target while ignoring false human targets. The following images show the performance achieved on test data. The first image below depicts an image of the tracked target (left), the test image ground truth mask (center), and the FCN's performance at labeling the test image (right). The second image shows the FCN's segmentation performance on non-targets. Again, the left image is the test image for non-targets, the center image is the ground truth mask, and the right image shows the FCN's performance at labeling non-targets. As is shown, most of the pixels are labeled correctly in both test images (top and bottom right) and hence the quad rotor will likely track the target well while ignoring false tracks.

![alt text][image6]
![alt text][image7]

### Network Architecture
#### 1. One of the primary goals of the project was to design and implemet a fully convolutional neural net. 

The fundamental concept is to train the network to label every pixel in an input image. The process labeling all pixes is called image segmentation. To do effective image segmentation, the network must first encode the image by learning a seriews of filters. Thus the convolutional stages of the network are used to learn image features by increasing the filter space. .The following table shows the network architecture. As can image shows the relevant DH parameters, joint angle directions and coordinate system axes.


![alt text][image8]

The DH table is shown next and is derived according the DH rules. Once the robot diagram is correctly annotated, the DH table follows in a straight forward manner. Note that one has to accound for the differences between the DH rules and the urdf file specification. This was a little difficult to grasp at first, but carefully following the lesson description was helpful.

Links | alpha(i-1) | a(i-1) | d(i-1) | theta(i)
--- | --- | --- | --- | ---
0->1 | 0 | 0 | .75 | q1
1->2 | -90 | .35 | 0 | -pi/2 + q2
2->3 | 0 | 1.25 | 0 | q3
3->4 |  -90 | -.054 | 1.5 | q4
4->5 | 90 | 0 | 0 | q5
5->6 | -90 | 0 | 0 | q6
6->EE | 0 | 0 | .303 | 0

#### 2. Using the DH parameters above, we can create individual transformation matrices about each joint. The individual joint transforms with the DH parameter substitutions are as follows:

T0_1 = [[cos(q1) -sin(q1) 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[sin(q1) cos(q1) 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[0 0 1 0.75] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[0 0 0 1]]
 
T1_2 = [[cos(q2 - 0.5*pi) -sin(q2 - 0.5*pi) 0 0.35] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 1 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [-sin(q2 - 0.5*pi) -cos(q2 - 0.5*pi) 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 0 1]]

T2_3 = [[cos(q3) -sin(q3) 0 1.25] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [sin(q3) cos(q3) 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 1 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 0 1]]
 
T3_4 = [[cos(q4) -sin(q4) 0 -0.054] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 1 1.5] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [-sin(q4) -cos(q4) 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;       [0 0 0 1]]
 
T4_5 = [[cos(q5) -sin(q5) 0 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [0 0 -1 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [sin(q5) cos(q5) 0 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [0 0 0 1]]
 
T5_6 = [[cos(q6) -sin(q6) 0 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [-sin(q6) -cos(q6) 0 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [0 0 -1 0] <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;      [0 0 0 1]]
 
T6_Grip = [[1 0 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;          [0 1 0 0] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;          [0 0 1 0.303] <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;          [0 0 0 1]]
 
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



![alt text][image5]
