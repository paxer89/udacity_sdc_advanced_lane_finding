## Writeup Project 2

---
## Update for resubmission

In my first submission i had an constant error of about 1.5m in my calculation of the offset of the car to the lane center.

The problem was that in the following line of `measure_curvature_real()`

```python
    #calculate offset in pixels
    offset_pix= warped.shape[0]/2-lane_center_pix
```
 I calculated with `warped.shape[0]` wich was the height of the warped image instead of the width of the warped image.
 With changing the line to

```python
    #calculate offset in pixels
    offset_pix= warped.shape[1]/2-lane_center_pix

```
I now get much more plausibe values for the lane offset.

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

[image1]: ./output_images/undistortedcalibration12.jpg "Undistorted Calibration Image"
[image2]: ./output_images/undistortedtest1.jpg "Undistorted test image"
[image3]: ./output_images/combined_binarytest2.jpg "Binary Example"
[image4]: ./output_images/warped_test2.jpg "Warp Example"
[image5]: ./output_images/sliding_windows_test2.jpg "Sliding window approach"
[image6]: ./output_images/around_poly_test2.jpg "Search around polynomial"
[image7]: ./output_images/projected_test2.jpg "Output"
[video1]: ./project_video.mp4 "Video"


## Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./Project2.ipynb" 

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Now I simply used the parameters calculated above with the calibration images on the test images, like here:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

To create a binary of potential pixel of lane lines I first defined different for threshholds and gradients:

`hls_s_select` to apply a threshold on S channel of HLS,
`hls_l_sobel_select` to apply a sobel filter in x direction on L channel of HLS,
`sobelx_grayscale` to apply a sobel filter in x direction on the grayscaled image,
`mag_thresh` to apply threshold on magnitude of a sobel filter in x, y and xy direction on the grayscaled image.

After trying out a lot of combinations and thresholds I ended up with a combination of `sobelx_grayscale` with thresholds `(20,100)` and `hls_s_select` with thresholds `(170,255)`. Pretty close to what was done in the lessons before but all other attempts were less satisfying than that.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

To warp the binaries to bird eye view I first undistorted the test image `straight_lines1` and manually measured the four source points:

| Source        |
|:-------------:|
| 595,450       |
| 686,450       |
| 1071,706      |
| 231,706       |


For the destination points i defined an offset of 0.25 of the image width and defined the points like here:

```python
dst = np.float32([[offset, 0],[width - offset, 0],[width - offset, height],[offset, height]])

```


From the source and the destination points I calculated the warping matrix and stored it and its inverse `M_warp` `M_warp_inv`.

With `M_warp` i warped the binary images like here:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?


To identify left and right lane line pixels and fit a 2nd order polynomial I defined two different functions `fit_polynomial_windows` with a sliding window approach and `search_around_poly` to search around an already given polynomial.

In `fit_polynomial_windows` I started with a histogram of the lower half of the warped binary and took the argmaxes of the left half and the right half for the starting points for windows for the left and right lane line. 

For a defined number of 9 rows I identified the nonzero pixels within the window and if a minimum number of pixels was reached the x position for the windows for the next rows were adjusted to mean x posiotion of the identified nonzero pixels. 
Resulting example: 

![alt text][image5]

Afterwards I fitted a second order polynomial on the identified nonzero left and right lane pixels.

In `search_around_poly` the nonzero pixels for left and right lane the are searched withing a margin left and right of a given polynamial of the previous frame. A new polynomials is fitted to the found pixels in the same way as in the function above.

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In `measure_curvature_real` I calculated the curvature of the fitted lane and the offset of the car to the middle of the Road.
First I defined the scaling factors `ym_per_pix = 30/720` and `xm_per_pix = 3.7/640`. I calculated the average of the polinomial factors of the left and the right lane. I calculated adjusted parameters including the scaling factors like here:

```python
    fit_adj[0] = mid_fit[0] * xm_per_pix/ (ym_per_pix**2)
    fit_adj[1] = mid_fit[1] * xm_per_pix/ym_per_pix
    fit_adj[2] = mid_fit[2]

```
With the formula given in the lesson I calculated the curvature radius and selected the raius on the lowest end (y_max) of the image.

For the offset I added the x position of the left and the right lane line at y_max and divided it by two. Subtracting this from `image_shape/2` results in the offset in pixels. With the scaling factor I calculated the offset in meters.

I projected both values directly on the undistorted original image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

In the function `project_lanes` I projected the detected lane back on the undistorted original image with the earlier calculated matrix `M_warp_inv`:

![alt text][image7]

#### Sanity checks

In `check_lane_sanity` I checked if the first and second pameters of left and right fitted polimnomials are similar within a certain threshold and if the lane width is within a given range of pixels. If any of this conditions is not given the lane sanity is returned as `False` otherwise as `True`. 

---


### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

In the video pipeline I performed the following steps:

* Undistorting the frame with the parameters caculated from the calibration images 
* Calculate a binary of potential lane pixels 
* Warp the binary to birds eye view
* Identifying lane pixels and fitting a polynomial
  * For the very first frames with the sliding windows approach
  * For later frames searching around the previous (smoothened) polynomial
  * If lane sanity check was false for n times in a row restarting with sliding windows approach
* Smoothening the fitted polinomial over the average of the last m frames
* Projecting the lanes on the undistorted frame
* Calculate and print the curvature and offset on the projected image


Here's a [link to my video result](./solved_project_video.mp4) on the project video.

Attemps on the challange videos didnt work out too well.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

* First of all I could speed up my pipeline a lot by:
  * Kicking out a lot of plotting and visualization steps out of the functions which were used to visualize the single steps but aren't necessary for the overall pipeline
  * Not calculating stuff like image shape and `plot_y` and so on over and over again but store them in an overall class
* The combination of threshold and filtering for potential lane line pixels is not realy satisfying yet. For the challange viedeos I would have to work on that.
* My attemps to implement an mechanism to prevent sliding windows losing lines in strong curves didn't work out. I could further work on that.
* My values for curve radius and especially the offset aren't completely plausible.
  * My chosen image points to warp the images aren't completely symmetrical to the middle of the image. In calculating the offset that should be considered.
  * The scaling factors `ym_per_pix` and `xm_per_pix` could be tuned further.
* My lane sanity checks and the smoothening of the fitted lines aren't really mature yet
* Lots of other stuff to improve :)

