# Udacity CarND: Advanced Lane Finding Writeup
#### Jon Wetherbee, September 2017 Cohort

---

## **Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to identify the left and right lanes in a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1a]: ./output_images/orig_calibration1_360.jpg "Original Calibration Image"
[image1b]: ./output_images/undist_calibration1_360.jpg "Undistorted Calibration Image"
[image2a]: ./output_images/orig_test6_360.jpg "Original Image"
[image2b]: ./output_images/undist_test6_360.jpg "Undistorted Image"
[image3]: ./output_images/threshold_test6_640.jpg "Binary Threshold Image "
[image4a]: ./output_images/warp_pts_straight_lines1_640.jpg "Undistorted Image"
[image4b]: ./output_images/undist_test6_360.jpg "Undistorted Image"
[image4c]: ./output_images/warp_test6_360.jpg "Warped Image"
[image5]: ./output_images/warp_plot_test6_720.jpg "Fit Visual"
[image6]: ./output_images/annotated_test6_640.jpg "Annotated Image"
[video1]: ./project_video.mp4 "Video"

### [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

---

## Writeup / README

This document describes the process involved in analyzing a video taken from a camera mounted on a car, and deriving the lane boundaries of the car while it is driving. 

The result of the coding portion of the project is an annotated video, where the lane, bounded by lane lines on either side, is superimposed in green on the original video frames. Additionally, the calculated values for the location of the car with respect to lane center, and the curvature radii of the lane, are written to each resulting frame in the output video.

The code for this project can be found in this Jupiter notebook: [CarND-Advanced-Lane-Lines-checkpoint.ipynb](https://github.com/jwetherb/CarND-Advanced-Lane-Lines/blob/master/CarND-Advanced-Lane-Lines.ipynb).

### Camera Calibration

Most commonly-used cameras introduce distortion in various ways. Fortunately there are ways of detecting this distortion and derive a matrix that can be used to transform the original image into an undistorted state. For this project, we were given twenty or so photos of a chessboard, from different angles and distances. These photos had been taken with the same camera that was on the car. Since we know the actual dimensions of an undistorted chessboard, we were able to use some opencv library methods to a) identify the corners of the chessboard in the original, distorted image, and then b) compare them to the corners on a known, undistorted chessboard, to produce distortion coefficients for calibrating the camera.

Once calibrated, I was able to apply the same anti-distortion transformation to each frame of the video, correcting each image prior as a first step in the process.

The code for this step is contained in the first code cell of the IPython notebook, identified by "Code section 1".

I began by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time we successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

In "Code section 2" I show an example of applying this distortion correction to a calibration test image (below, left) using the `cv2.undistort()` function and obtained this result (below, right): 

!["Original"][image1a] !["Undistorted"][image1b]

## Pipeline (single images)

This section describes the steps I used to detect the lane lines in an already undistorted image, like the one above. This process involves the following steps:

1. Undistort the image, using the calibration coefficients obtained in the previous section
2. Apply binary threshold transforms to identify the path of the lane lines
3. Warp the image so that we have a "birds-eye" view to work with
4. Calculate the curvature of the lane lines, from this birds-eye view, as a polynomial function
5. Project this polynomial -- also known as the "lane fit" -- back onto the warped image
6. Unwarp the "lane fit" lines and overlay them on top of the original (undistorted) image, to demonstrate that they are correct
7. Finally, derive a solid green polygon that lives between the lane lines, and superimpose it on top of the original image, showing the full breadth of the lane

### Example of a distortion-corrected image

To demonstrate the process, in "Code section 3" I apply the distortion correction to one of the test images. On the left is the distorted original, and on the right is the same image after having undergone distortion correction:

![alt text][image2a] ![alt text][image2b]

### Creating the thresholded binary image

Once I had corrected the image to overcome camera distortion I applied several binary threshold algorithms to enhance the lane lines and isolate them in the image.

**All code lines in this section refer to "Code section 4" in the Jupyter notebook.**

#### Sobel Gradient Threshold

I converted the image to grayscale and experimented with sobel gradient transforms which identify horizontal and vertical gradients in the image. This proved useful, particularly the x gradient, which identifies pixels that are part of a vertical line. These pixels, which indicate a more-or-less vertical line, are useful in identifying lane lines which generally appear in a vertical (sloping) orientation. The code for this transform is in lines 28-50, and the method is called from apply_threshold_transforms() and tested in lines 117-181.

I also experimented with several of the other sobel thresholds -- magnitude and direction. I tried various threshold parameters, but did not find any that helped elucidate the lane lines in my test images, so I excluded their input from my final binary image. The code for these transforms is in lines 56-93.

#### Color Threshold

Splitting the original (undistorted) color image into its three color channels, and isolating each one in turn and applying a threshold, turns out to be a very useful step in identifying the lane lines. I tested two forms of color processing -- splitting out the BGR (Blue, Green, Red) channels and identifying HLS (Hue, Luminance, Saturation) levels -- and applied threshold limits. What seemed to work best was using the Red color channel combined with the Saturation level (see lines 138-142)

#### Polygon mask

Using the knowledge that the lane lines will generally be isolated to the bottom half of the frame, and within a trapezoidal section, we can apply a mask to this area and ignore the rest of the image. This step is applied by calling `apply_trapezoid_mask()`, defined in lines 11-23.

#### Resulting binary image

Here's the result of combining the areas of the image that were highlighted by an OR combination of the sobel or color threshold processing, and then applying the polygon mask. Lines 117-166 of "Code section 4" apply the threshold algorithms that produce the binary image below.

![alt text][image3]

### Creating the perspective transform

Once we have a thresholded binary image that isolates the lane line information, I warp it into a "birds-eye" view for further processing. The perspective warping matrix M, and its de-warping inverse matrix Minv, are derived in `calc_perspective_transforms()` in "Code section 5".

The values for providing the perspective transform were arrived at by applying a polygon to one of the example images, and then adjusting the vertices until it matched the lane on my test image as it receded into the horizon. The trapezoidal polygon has at its base the distance between the two lane lines in the foreground (bottom of the frame), and at its top the distance between the lane lines in the background (middle of the frame). Since we know the lane is the same actual width in both places, we can use these points to "warp" the image into a top-down "birds-eye" representation where the lane is shown with a fixed width.

The trapezoid is shown superimposed on my straight-line test image, here:

![alt text][image4a]

The values of my polygon turned out to be:

```python
    src_pts = np.float32([[571, 468],  [714, 468], [1112, 720], [200, 720]])
    top_left = [320, 200]
    top_right = [920, 200]
    bottom_right = [920, 720]
    bottom_left = [320,720] 
    dst_pts = np.float32([top_left,top_right,bottom_right, bottom_left])
```

This resulted in the following source and destination points:

| Position        | Source        | Destination   |
|:----------------|:--------------|:--------------| 
| Top left        | 571, 468      | 320, 200      | 
| Top right       | 714, 468      | 920, 200      |
| Bottom right    | 1112, 720     | 960, 720      |
| Bottom left     | 200, 720      | 320, 720      |


I verified that my perspective transform was working as expected by viewing my test image both before and after the "warp" perspective transformation, and verifying that the lane lines are parallel in the resulting transformation.

![alt text][image4b] ![alt text][image4c]

### Identifying the lane lines

Using the new warped "birds-eye" binary image (generated in "Code section 6") with the lane lines identified, I identified a handful of points on both the left and right lane lines that could be mapped to a linear function, f(y) = A * y**2 + B * y*y + C * y. The process of identifying these points was really the crux of the project, and there were two ways of arriving at these points: one slower but more thorough, and the other faster because it leveraged information derived for the previous frame in the video.

For the first method, shown in method `calc_line_fit_with_sliding_boxes()` in "Code section 7", I began by dividing the warped binary image into 9 horizontal slices. Using a convolutional technique of searching for the lane lines within horizontal slices of the image, I leveraged the knowledge that a lane line is contiguous, and so from one horizontal slice to the next I could assume the line would continue near where it left off in the previous slice.

1. First I found all of the nonzero points in the bottom half of the warped binary image
2. I created a histogram of these points, and identified the peak values on both the left and right side of the histogram
3. For starting position of the first (bottom-most) left and right boxes, I used the peak values found in the histogram
4. For the nine slices I created a box on each side of the image, and appended all of the nonzero points within that box to my list of points for both the left and right side
5. To place the boxes for each new slice, I centered the box on the average x value for the points found in the previous box
6. Once I had collected all of the points that tracked the left and right lane lines, I called `np.polyfit()` to calculate the fit:

```python
    # Fit a second order polynomial to each
    left_fit = np.polyfit(lefty, leftx, 2)
    right_fit = np.polyfit(righty, rightx, 2)
```

#### Putting it all together

Applying these steps to my test image, and showing the sliding boxes and the resulting "fit" lines, is performed in "Code section 8". The result is shown here:

![alt text][image5]

### Peforming a "fast fit" to generate subsequent lane fits

As mentioned in 'Identifying the lane lines' above, I also use a second, faster, method to derive the lane lines. This technique can be used when processing a contiguous video stream, and only requires that a lane line was successfully found on the previous frame in the video. Since this is typically the case, this approach is used for processing a majority of the line fits in a video stream.

The `fast_fit()` method is defined in "Code section 10". I begin with the left and right lane line fits from the previous frame, and find all of the nonzero points in the current image that are within 100 pixels on either side of the previous frame's lane lines. Once these points are derived, I call `np.polyfit()` again with point sets for the left and right lines to detect the lane fits for the current image.

### Calculating the lane curvature and lane offset

In "Code section 11", in method `calc_lane_curvature()`, I calculate the lane curvature for each frame. The principal inputs to the `calc_lane_curvature()` method are the two line fits which are adjusted from pixels to real-world meters and used to calculate the curvature of the lane at the very bottom of the frame.

In "Code section 12", method `calc_vehicle_offset()`, I calculate the car's position in relation to lane center, in meters. Again, using the bottom of the frame as a reference, I calculate the number of pixels between the left and right lanes, calculate the distance between frame center and lane center, convert from pixels to meters, and arrive at my current offset from lane center at the moment that image was recorded.

### Projecting the lane back onto the original image 

As a final step in the pipeline, in "Code section 13", I write the resulting information back onto the frame being processed, so it can be saved as an annotated video frame.

The result of our test image is shown here:

![alt text][image6]

---

### Pipeline (video)

Here is a [link to my video result](./project_out.mp4)

---

### Discussion

This project offered the possibility of using many techniques to enhance the video images to illuminate the lane lines, and also to process the result to calculate the lane fit. The main problem I faced was there were a lot of options to consider (and only so much time with which to consider it)! It was interesting to experiment with using different threshold techniques on the binary image, and I wish I had more time to test different threshold levels and combinations of transforms. 

Although my result performed admirably for the base project video, it fell down a few times on the challenge and especially the harder_challenge videos. I'm sure the additional false-line noise (especially road shoulder barriers that mimic the characteristics of lane lines), and shadows and other images on the road, could all be handled through a more thorough use of sobel and color threshold combinations. 

Also, I thought about (but didn't deploy) a strategy of ensuring that the lanes remain parallel when faced with noisy or absent lane information on one side or the other. Similar to how my `fast_fit()` method leveraged knowledge of the previous frame's line fits, whenever a lane line is not clearly identified on once side, it would be relatively easy to check whether the lane on the other side of that bit of road (y-val) was well established. And if so, extrapolate the missing lane info to be a lane-width apart from the well-known position of the other lane.
