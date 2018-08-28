# Advanced Lane Finding Project

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

[image1]: ./output_images/undistort_output.png
[image2]: ./output_images/perspective_transform.png
[image3]: ./output_images/binarization.png
[image4]: ./output_images/histogram.png
[image5]: ./output_images/sliding_window.png
[image6]: ./output_images/search_from_prior.png
[image7]: ./output_images/fit.png
[image8]: ./output_images/result.png

[image9]: ./output_images/wrong_line.png
[image10]: ./output_images/wrong_line_histogram.png
[image11]: ./output_images/wrong_line_corrected_histogram.png
[image12]: ./output_images/wrong_line_corrected.png

[video1]: ./output_videos/project_video.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image. After a successful detection of chessboard corners, I append a duplicate of these real-world coordinates to the `object_points` array and positions of corresponding points in the image to `image_points.`

I then used the output `object_points` and `image_points` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.

### Pipeline

#### 1. Provide an example of a distortion-corrected image.

As a first step of the pipeline, I correct for distortion with `cv2.undistort()` function using the camera matrix obtained during calibration process. Here's an example of such correction applied to a test image:

![alt text][image1]

#### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

_The code for this step is contained in the fourth code cell of the IPython notebook located in `./solution.ipynb`._

To calculate the transformation matrix, I first used one of the test images to locate four points with known relative positions in the real world. I chose the image such that the car was driving in the center of a straight lane and a segment of a white dashed line was clearly visible. Having known the length of such a segment to be about 3 meters and the width of the lane to be about 3.7 meters, I then calculated rectified positions of those points, preserving the relation to real-world distances as `SCALE_X` and `SCALE_Y` constants.

I then used used the transformation matrix to create a birds-eyed view for each image in the pipeline, as shown below. The points used to calculate the transformation matrix are marked as well.

![alt text][image2]

#### 3. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

_The code for this step is contained in the fifth code cell of the IPython notebook located in `./solution.ipynb`._

To binarize the image, I used a combination of two thresholds; one based on saturation an the other based on gradient magnitude in the x direction. I purposely put this step **after** perspective transform, so that the y axis in the image would be parallel to the vehicle's direction, which in turn improved robustness of the gradient operation. Here's an example of such a binarization:

![alt text][image3]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

_The code for this step is contained in the sixth to ninth code cells of the IPython notebook located in `./solution.ipynb`._

I used two techniques to detect lane line pixels and used them interchangeably depending on whether the lane was correctly identified in a previous frame.

First approach (used when there is no previous detection) consisted of two steps, the first being calculating a column-wise histogram on the bottom half of the image. In the histogram, a value for each bin represented how many white pixels were present in the corresponding columns. I then identified two peaks, one for each lane line.

![alt text][image4]

Next, starting from the peaks, I used a *sliding window* technique to identify pixels corresponding to each line. An example is shown below.

![alt text][image5]

If previous detection data was available, instead of the two steps mentioned earlier I used *search from prior* technique to identify the lane line pixels. Here I select all the white pixels lying within a specified margin of the previous fit.

![alt text][image6]

Regardless of how the pixels were detected, the last thing to do is fit two second-degree polynomials to those points using `np.polyfit()` function. A visualization of the final result is shown below.

![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

_The code for this step is contained in the tenth and eleventh code cells of the IPython notebook located in `./solution.ipynb`._

The calculations for radius of curvature are based one the [radius of curvature equation](https://www.intmath.com/applications-differentiation/8-radius-curvature.php) and the distance from center is measured from the offset of the midpoint between the two lines in respect to image center. In both cases the scaling factor introduced earlier is taken into account to properly convert the values to meters.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One of the problems I faced was incorrect detection of lane line pixels. Due to the changing environment, including varying lightness and passing cars, it's a challenging task to make the binarization step robust enough to filter out all unwanted components. This in turn sometimes results in erroneous (...). An extreme case of such event is shown below.

![alt text][image9]

The problem here was that As you can see, not only did the algorithm falsely recognize a passing car as part of the lane, the whole sliding window for the right lane line started in the wrong place. The reason for this can be found when investigating how the histogram was calculated from the bottom half of the image.

![alt text][image10]

There are two lane lines visible on the right side of the image, one for the lane the car is currently on and one for the next lane. Because a bigger part of the second one is visible on the screen, that's where the largest peak appears. That's also where the sliding window will start.

The solution I came up with for this issue was clamping the histogram within a specified range. Starting from the center of the image, I only considered white pixels within one lane width to the left and right. Here's how the resulting histogram looks like:

![alt text][image11]

And here's the effect of this fix on the whole pipeline:

![alt text][image12]

However, even though the detection works much better than before, it can still suffer from imperfect binarization. For example, problems could occur if there was a car directly before the camera, or even changing lanes.
