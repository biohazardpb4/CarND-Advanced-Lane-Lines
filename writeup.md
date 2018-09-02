## Advanced Lane Finding Writeup

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
[image1]: ./output_images/undistorted.png "Undistorted"
[image2]: ./output_images/road_transformed.png "Road Transformed"
[image3]: ./output_images/binary_filter.png "Binary Example"
[image4]: ./output_images/warped.png "Warp Example"
[image5]: ./output_images/color_fit_lines.png "Fit Visual"
[image6]: ./output_images/output.png "Output"
[video1]: ./test_videos_output/project_video_output.mp4 "Video"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook located in "./advanced_lane_finding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in cell 6, functions `filter_gradient` and `filter_color`). To tune these parameters, I created interactive IPython functions called `interactive_hls_filter` (cell 4) and `interactive_sobelx_filter` (cell 5). I iterated a bit on the color filter, introducing a lightness filter when saturation alone was not sufficient to detect lanes covered by shadows. Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in cell 7.  The `warp()` function takes as inputs an image (`img`).  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32([[192, 720], [576, 460], [703, 460], [1089, 720]])
dest = np.float32([[300, 720], [300, 0], [950, 0], [950, 720]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 192, 720      | 300, 720      |
| 576, 460      | 300, 0        |
| 703, 460      | 950, 0        |
| 1089, 720     | 950, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To identify lane pixels within my code (`find_lane_pixels` in cell 8), I took the sliding window approach. It begins by cropping the image to the bottom half since lanes typically only show up there unless you're driving a rally car. The supplied image is expected to be binary and warped into a top down view. A histogram is taken along the x-axis, then the image is split in half. The histogram frequency max of the left side is taken to mean where the left lane is and the same is done for the right lane. That is used as a starting point for the lanes. Then the vertical space is divided into `nwindows` windows. For each window, the average of the x coordinates of the activated pixels within it is taken, and that average is used to determine the x coordinate of the following window. Any pixels that end up falling within these windows is stored as a point along the lane. Only if there are `> minpix` pixels within a window does this window influence the x coordinate of the following window.

Fitting the activated pixels to a polynomial is done in `fit_polynomial`, also in cell 8. It uses the `np.polyfit` function to determine the estimated equations of the left and right lanes. It estimates equations in pixel and real world coordinate systems, so that we can draw and provide diagnostics to the end user, respectively. As an attempt to correct for detected lanes jumping around a bit in the test videos, a memory of previously computed polynomials is stored in `memory`, and the median of those coefficients is used as the returned output.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature of the lane is computed in `measure_curvature_meters` in cell 9. It uses a static y-coordinate of `1200*30/720`, which represents the bottom of the image in meter-space, and uses the formula `((1 + (2*a*y + b)**2)**(3/2))/abs(2*a)` to compute the radius of curvature at bottom of the image (aka where the camera is on the car's dash). Even though one would expect both lanes to be parallel, that may not always be the case, so both left and right lane curvature are averaged.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Using a combination of `display_curvature` in cell 9 and `display_lane` in cell 10, `display_lane_pipeline` results in displaying the detected lane back onto the original image.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

To approach this problem, I began by writing down all of the steps that the pipeline ultimately needed to do in comments. Then step by step, I created pipelines which processed all supplied test images with all steps that were up until that point, making partial pipelines. This piece-by-piece approach helped me debug and improve individual steps of the pipeline. Also, for color and gradient filtering where I hard-coded arbitrary thresholds, I used an IPython interactive function to help determine useful values. This isn't as great as being fully automated, but is much better than plugging in static values and waiting for the result after every little tweak.

To detect lane markers and display them, along with diagnostic information like lane curvature and position, I took several steps. I started by calibrating the camera being used by using supplied pictures of known planar coordinates (chess boards) from many angles. Once the camera's distortion coefficients were known, I undistorted the supplied road images. After that, I combined a color filter in the HSL color space and an x-gradient filter to pick out lane pixels. Using detected pixels, lane equations were estimated were established. After that, lane estimates were drawn onto a warped original image and then unwarped. Diagnostic information was also determined via the estimated line equations and displayed on the final image. To enhance robustness, I stored the last 5 estimated line equations for each lane, and used the median of them as the current line equation.

Camera calibration appeared to work without much work on my part. Very nice calibration facilities have been built into OpenCV which allow one to feed in pictures of chess boards and get back image coordinates, which can easily be used for calibration. Filtering by saturation and lightness worked well because lanes tend to use heavily saturated colors so that they are visible. Lightness worked because shadows also appeared to produce highly saturated dark gray colors on certain lanes. Warping into a top-down view worked well for line equation estimation because one can consider the line in two dimensions instead of three, which greatly simplifies reasoning. Storing the past few equation estimates and operating based on the median of those smooths out lane estimates by reducing estimate variance, since single samples have less power to affect individual estimates.

The pipeline fails under lighting condition changes, like in the challenge video. It also likely fails at recognizing sharp curves or road segments that curve multiple ways quickly, but it's tough to tell because the algorithm is not good enough to see past lighting differences.

To improve this project further, I would first split the challenge video into frames, then take the underlying images from the worst performing frames and tune the binary color and gradient filters such that the lighting differences can be handled. Then I would also make a mechanism that only displays the lane curve for as long as it fits the lane. If the lane goes in another direction (say an S-curve), then this algorithm would only output a short lane segment for the immediate curve. Steering wheels are most concerned with immediate turning angle, so this is probably more important than doing something like estimating the entire lane using higher order polynomials (or piece-wise functions) that can capture that curvature.
