
**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[distortedImage]: ./camera_cal/calibration1.jpg "Distorted"
[undistortedImage]: ./output_images/undistorted.jpg "Undistorted"
[left]: ./left.jpg "Road Transformed"
[binary_analysis]: ./output_images/binary_analysis.jpg "Binary Analysis"
[binary_analysis_left]: ./output_images/binary_analysis_left.jpg "Undistorted"
[transform_src]: ./output_images/straight_transform_src.jpg "straight transform source"
[transform_dest]: ./output_images/straight_transform_dest.jpg "straight transform destination"
[lane_marks]: ./output_images/lane_marks.jpg "lane marks"
[project_detected_lane]: ./output_images/project_detected_lane.jpg "projected detected lane"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first section "Camera Calibration" of the IPython notebook located in "./P4-Advanced Lane Detection.ipynb" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

orig image:
![alt text][distortedImage]

undistorted image:
![alt text][undistortedImage]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][left]
####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
For code please see section "Thresholding" of the notebook.

I used a combination of color and gradient thresholds to generate a binary image.  After some trials including challenge video frames, I realised this is a short blanket problem and tweaking params to fix a frame breaks others.  I devised a function that analyses multiple frames and display the intermediate steps (limited to 7 horizontal images or they would become too small).  The end result is combination of L and S thresholds in HLS image and a combined magnitude & direction gradient thredholds.  The conclusion is that 2 out of 3 (H, L, magnitude & direction) are usually enough to cleanly identify left & right lane markings but sometimes the third introduces noise.  I added some simple logic to ignore the bad component based on mean L or S value for the whole frame.  The logic here is if too much 'energy' is picked up, we better off without this component.  To continue this work, I could calculate number of white pixels after threshold for each component and if that value is beyond a threshold value it means too much noise is going to enter the binary output frame. Images named bad_imageX.jpg are images from the video that I failed to detect lane so I save them for further analysis in future pass.

![alt text][binary_analysis]

Below is thresholded binary image of the example above:

![alt text][binary_analysis_left]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

For code please see section "Perspective Transform" of the notebook.

I first calculated pespective transform M based on these hard coded source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 30, 720       | 0, 720        | 
| 550, 465      | 0, 0          |
| 730, 465      | 1280, 0       |
| 1250, 720     | 1280, 720     |

The code for my perspective transform includes a function called `warper()`.  The `warper()` function takes as inputs an image (`img`), as well as perspective transform (`M`). 

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image (where the road is traight) and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][transform_src]


![alt text][transform_dest]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

For code please see section "Finding the lane marks" of the notebook.

To detect lane polynomial lines I followed these steps (see `detect_lines(binary_image)` function):

1. take bottom half of image and calculate histogram of number of white pixels for each collumn.
2. Split histogram to 2 parts (left, right) and determin max value of each half.  This is roughly where each lane appears at bottom half of the image
3. split image again, this time to 9 horizontal strips.  Start from bottom and for each strip, consider a rectangle with height of the strip, centered around histogram determined horizontal centers.  The width is 100 pixels to right / left of the center.
4. mark pixels white pixels from binary image that are enclosed in this rect and adjust historgram centers to mean horizontal of the strip's white pixels. 
5. Repeat for next strip all the up to the top strip

Once finished all strips, use numpy.polyfit to fit polinomial close to the marked white pixels.

Here is example output of this step:

![alt text][lane_marks]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

##### Radius of Curvature
For code please see section "Finding the lane marks" of the notebook.

This is also done as part of `detect_lines(binary_image)`. 
Here are the steps:

1. generate y coordinates for all points in the y axis (i.e. height of image)
2. calculate x coordiate for each y value
3. So far these are in units of pixels.  I convert both (y and x value) to meters by multiplying by pixel / meter ratio.  I can calculate the x axis ratio by multiplying lane width (3.7m) by distance in pixels at bottom of image.  Y axis ratio is calculated similarly.  I assume the projected area is 30 m like in-class example.
4. calculate radius R by formula provided in class

##### position of vehicle

For code please see section "Lane offset" of the notebook.

From polymonial formulas I recalculate x-axis position for each lane at bottom of screen.
Based on difference between them, I recalculate pixel / meters ratio, take difference midpoint of these points to middle of image where I expect the middle to be if the car was exactly in the middle, and convert the difference to meters.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

For code please see section "Project back to camera image" of the notebook.

From polynomial formulas, calculate (x,y) points and plot with `vc2.fillPoly()`.  I then unwarp the image and combine on top of original undistorted image.



![alt text][project_detected_lane]


---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The hardest part is determining steps & thresholds for converting image to binary with only distinctive lane markings showing.  With more time I would concentrate on improving that.  As discussed in ection 2 above, smarter decisions on which  binary image 'factors' to use will make the result more rebust.  This is where my solution fails to distinguish lane markings from shadows and other signals as experienced in challenge videos.

