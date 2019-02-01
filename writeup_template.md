## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

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

[image1]: ./output_images/undistort20.jpg "Undistorted"
[image2]: ./output_images/test_undistort3.jpg "Road Transformed"
[image3]: ./output_images/color_binary2.jpg "Binary Example"
[image4]: ./output_images/warped2.jpg "Warp Example"
[image5]: ./output_images/measured2.jpg "Fit Visual"
[image6]: ./output_images/test1.jpg "Output"
[video1]: ./output_videos/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it! The code for all the steps is contained in the IPython notebook located in "./examples/example.ipynb".  

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

Code in second block of "example.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Code in the end of second block of "example.ipynb".

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

I used `cv2.calibrateCamera` and put `objpoints`, `imgpoints` and image shape to get the image correction matrices. Then I used `cv2.undistort` and put correction matrices and images to get the distortion-corrected image.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Code in the third block of "example.ipynb".

I first did a Gaussian smooth of radius 5. Then I select the line pixels using 30-80 x-gradient value combined with an intersection of 150-255 saturation channel and 0-30 hue channel in HSL color space.

Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Code in the fourth block of "example.ipynb".

I hardcoded the source and destination points in the following manner:

```python
# manually set warp points
src_points = np.float32([[554,480],  [305,659], [1016,659], [736,480]])
dst_points = np.float32([[300,0], [300,720], [980,720], [980,0]])
# get transform matrix
M = cv2.getPerspectiveTransform(src_points, dst_points)
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 554, 480      | 300, 0        | 
| 305, 659      | 300, 720      |
| 1016, 659     | 980, 720      |
| 736, 480      | 980, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Code in the fifth block of "example.ipynb".

I used 9 sliding windows with 100 width and 50 pixels as centering threshold to select two lines' pixels. I tried to find line pixels based on previous polynomial, but it worked terrible, so I gave up using that.

After two line pixels are found, I fit a second order polynomial using `np.polyfit`. Then I construct a fitted line to color the green block of the path.

The line pixels are colored either red or blue based on left or right line. The area between fitted lines is colored green.

```python
## Visualization ##
# Create an output image to draw on and visualize the result
out_img = np.dstack((binary_warped, binary_warped, binary_warped))
# Colors in the left and right lane regions
for i in range(out_img.shape[0]):
    out_img[i, int(round(left_fitx[i])):int(round(right_fitx[i]))] = [0,255,0]
out_img[lefty, leftx] = [255, 0, 0]
out_img[righty, rightx] = [0, 0, 255]
```

The result image:

![alt text][image5]



#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Code in the fifth block of "example.ipynb".

Using the two fitted lines to calculate radius of both line. The displayed radius is the average of two lines.

```python
# calculate radius of curvature
    y_eval = binary_warped.shape[0]
    left_curverad = (1+(2*left_fit[0]*y_eval*ym_per_pix+left_fit[1])**2)**1.5/abs(2*left_fit[0])  ## Implement the calculation of the left line here
    right_curverad = (1+(2*right_fit[0]*y_eval*ym_per_pix+right_fit[1])**2)**1.5/abs(2*right_fit[0])  ## Implement the calculation of the right line here
    mean_curverad = (left_curverad + right_curverad) / 2
```

The position of the vehicle with respect to center is calculated as the following:

```python
# calculate vehicle pos to line center
# negative means image center is on the right of line center
dist_center = ((leftx[0] + rightx[0])/2 - binary_warped.shape[1]/2)*xm_per_pix
```

It is the difference between image center x coordinate and two found bottom pixels' average x coordinate.

The pixel to meter conversion is hardcoded. The road line block is 3 meters long corresponding to 175 warped pixels. The distance between left and right line is 3.7 meters long corresponding to 680 warped pixels.

```python
ym_per_pix = 3/175
xm_per_pix = 3.7/680
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Code in the sixth block of "example.ipynb".

I used `cv2.putText` to display radius of curvature and position of vehicle w.r.t center.

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](../output_videos/project_video.mp4)

Here's a [link to my harder challenge video result](../output_videos/harder_challenge_video.mp4)

The pipeline is unable to process `input_videos/challenge.mp4`. There is an image where the pipeline can't find any line pixels and crash the process.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The pipeline is definitely not robust enough. It is a really hard project and need to test lots of parameters to make sure every step of the pipeline work. The pipeline will fail when there is too much light or there are some textures on the road. The pixels selection will often include either too much noise or no lines. Implementation of supervised machine learning will make it more robust.