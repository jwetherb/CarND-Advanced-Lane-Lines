## Udacity CarND: Advanced Lane Finding Writeup
#### Jon Wetherbee, September 2017 Cohort

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

[image0]: ./output_images/small_calibration1.jpg "Original Calibration Image"
[image1]: ./output_images/small_undist_calibration1.jpg "Undistorted Calibration Image"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./output_images/small_threshold_straight_lines1.jpg "Binary Example"
[image4]: ./output_images/small_perspective_straight_lines1.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

---

### Writeup / README

This document describes the process involved in analyzing a video taken from a camera mounted on a car, and deriving the lane boundaries of the car while it is driving. 

The result of the coding portion of the project is an annotated video, where the lane, bounded by lane lines on either side, is superimposed in green on the original video frames. Additionally, the calculated values for the location of the car with respect to lane center, and the curvature radii of the lane, are written to each resulting frame in the output video.

The code for this project can be found in a Jupiter notebook here: [CarND-Advanced-Lane-Lines-checkpoint.ipynb](https://github.com/jwetherb/CarND-Advanced-Lane-Lines/blob/master/CarND-Advanced-Lane-Lines.ipynb).

### Camera Calibration

Most commonly-used cameras introduce distortion in various ways. Fortunately there are ways of detecting this distortion and derive a matrix that can be used to transform the original image into an undistorted state. For this project, we were given twenty or so photos of a chessboard, from different angles and distances. These photos had been taken with the same camera that was on the car. Since we know the actual dimensions of an undistorted chessboard, we were able to use some opencv library methods to a) identify the corners of the chessboard in the original, distorted image, and then b) compare them to the corners on a known, undistorted chessboard, to produce distortion coefficients for calibrating the camera.

Once calibrated, I was able to apply the same anti-distortion transformation to each frame of the video, correcting each image prior as a first step in the process.

The code for this step is contained in the first code cell of the IPython notebook located in [CarND-Advanced-Lane-Lines-checkpoint.ipynb](https://github.com/jwetherb/CarND-Advanced-Lane-Lines/blob/master/CarND-Advanced-Lane-Lines.ipynb)

I began by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time we successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

In code step [2] I show an example of applying this distortion correction to a calibration test image (below, left) using the `cv2.undistort()` function and obtained this result (below, right): 

!["Original"][image0] !["Undistorted"][image1]

### Pipeline (single images)

This section describes the steps I used to detect the lane lines in an already undistorted image, like the one above. This process involves the following steps:

1. Undistort the image (covered above)
2. Apply color and sobel threshold transforms to isolate the path of the lane lines
3. Warp the image so that we have a "birds-eye" view to work with
4. Calculate the curvature of the lane lines, from this birds-eye view, as a polynomial equation
5. Project this polynomial -- also known as the "fit" -- back onto the warped image
6. Unwarp the "fit" lines and overlay them on top of the original (undistorted) image, to demonstrate that they are correct
7. Finally, derive a solid green polygon that lives between the lane lines, and superimpose it on top of the original image, showing the full breadth of the lane

#### Example of a distortion-corrected image

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

#### Creating the thresholded binary image

Once I had
I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
