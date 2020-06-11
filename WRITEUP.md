## Writeup CarND-Advanced-Lane-Lines

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

[image1]: ./output_images/original_undistorted.png "Undistorted"
[image2]: ./output_images/test4_undistorted.png "Test4 Undistorted"
[image3]: ./output_images/test4_thresh_S.png "Test4 Thresholded S"
[image4]: ./output_images/test4_thresh_R.png "Test4 Thresholded R"
[image5]: ./output_images/test4_thresh_sobel_L.png "Test4 Thresholded Sobel L"
[image6]: ./output_images/test4_binary_warped.png "Test4 Binary Warped"
[image7]: ./output_images/test4_undistorted_warped.png "Test4 Undistorted and Warped"
[image8]: ./output_images/test4_fitted_lane.png "Test4 Fitted Lane Pixels"
[image9]: ./output_images/test5_fitted_lane_dynamic.png "Test5 Fitted Lane Pixels to Priors"
[image10]: ./output_images/test5_lane_curvature.png "Test5 Lane Marker Plotted Back"


[video1]: ./output_images/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second and third code cell of the IPython notebook `Advanced_Lane_lines.ipynb` and is adapted from the [Udacity Nanodegree](https://github.com/udacity/CarND-Camera-Calibration).

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![Original Distorted Image and Undistorted Image][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The distortion coefficients can then also be used to undistort the test images like this one:
![Test4 Undistorted][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used color and gradient thresholds to generate a binary images (thresholding steps at code cell 7, 8 and 9 of `Advanced_Lane_lines.ipynb`).  Here's an example of my output for these steps.  

![Test4 Thresholded S][image3]
![Test4 Thresholded R][image4]
![Test4 Thresholded Sobel L][image5]

The thresholds are `min: 160, max: 255` in the S channel (HLS color space) and `min: 220, max: 255` in the R channel (RGB color space). Additionally, the sobel operator is applied in x-direction on the L channels of the HLS color space with thresholds `min: 20, max: 100`. 

The resulting binary is a logical "OR" link of those three binary images (done after the transform step mentioned below).

![Test4 Binary Warped][image6]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform to a top down view includes a function called `warp_image()`, which appears in code cell 10 of `Advanced_Lane_lines.ipynb`.  The `warp_image()` function takes as inputs an image (`img`), and applies a hardcoded perspective transform using the source `src` and destination `dst`points in the following:

```python
imshape = img.shape
y_offset = imshape[0] / 8
x_offset_top = imshape[1] / 29
x_offset_bottom = imshape[1] / 5.8
y_offset_bottom = imshape[0] // 30

src = np.array([[(imshape[1] / 2 - x_offset_top, imshape[0] / 2 + y_offset),  # top left
                 (imshape[1] / 2 + x_offset_top, imshape[0] / 2 + y_offset),  # top right
                 (imshape[1] - x_offset_bottom, imshape[0] - y_offset_bottom),  # bottom right
                 (x_offset_bottom, imshape[0] - y_offset_bottom)  # bottom left
                ]], dtype=np.float32)

offset = 350
dst = np.float32([[offset, 0],   # x, y
                  [imshape[1] - offset, 0],
                  [imshape[1] - offset, imshape[0]],
                  [offset, imshape[0]]])
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![Test4 Undistorted and Warped][image7]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then (code cell 12 of `Advanced_Lane_lines.ipynb`) I take the histogramm of the bottom half of the image and use the peaks as the starting points of the left and right lanes. Subsequently, a sliding window, placed around the line centers, is used to find and follow the lines up to the top of the frame. The sliding window has a width of +/- 100 pixels originating from the center of the peaks and the next window is recentered if at least 100 pixels are in found in the previous window. All pixels in the left windows are counted as left lane marking pixels and all pixels in the right windows are counted as right lane marking pixels. Then, I fit my lane lines with a 2nd order polynomial as shown below:

![Test4 Fitted Lane Pixel][image8]

In code cell 14 of `Advanced_Lane_lines.ipynb` I am demostrating an alternative approach using prior knowledge of the position of the left and right lanes to form the windows.

![Test5 Fitted Lane Pixel Dynamic][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

With the fits from above and knowledge about the dimensions of the warped image (3.7 m  is the width of the road, 3 m is the length of one lane marker) we can calculate the size of each pixel in x and y direction as in the following:  
```python
ym_per_pix = 28/720  # meters per pixel in y dimension
xm_per_pix = 3.7/580  # meters per pixel in x dimension
```

The radius of the left and right lane markings can then be approximated with 
```python
left_curverad = ((1 + (2*left_fit[0]*y_eval*ym_per_pix + left_fit[1])**2)**1.5) / np.absolute(2*left_fit[0])
right_curverad = ((1 + (2*right_fit[0]*y_eval*ym_per_pix + right_fit[1])**2)**1.5) / np.absolute(2*right_fit[0])
```

For the distance to the center I am assuming the camera to be mouted at the cente rof the car. Thus, the position of the car is in the center of the image
```python
pos_vehicle = imshape[1] / 2
```

After that, the distance to the center can be calculated by
```python
cm_per_pixel = 3.70 / 580 * 100
road_center = (left_fitx + right_fitx) // 2
distance_to_center = (pos_vehicle - road_center) * cm_per_pixel
```
with left_fitx and right_fitx being the pixel values of the lane marker function at the bottom of the image.

See the code in cell 15 of `Advanced_Lane_lines.ipynb` for the full functions. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code cell of `Advanced_Lane_lines.ipynb` in the function `unwarp_lane_lines()`. I also added text for the values in the function `add_text_to_image()` Here is an example of my result on a test image:

![Test5 Lane Marker Plotted Back][image10]

All other test images can be found [here](./output_images/)

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_videos/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

I had most difficulties with the sudden changes in brightness as in test images 4 and 5. Even with using the HLS color space and combining different thresholded binaries the results still were little unstable at those spots on the video when applying the pipeline to each frame.  So I decided to use the prior knowledge from previous frames to look for the lane markings. This already gave a great improvement but I also decided to average out outliers by weighted averaging the coefficients of the polynomial functions of the latest 7 frames. For that I created a new class `Pipeline` as shown in code cell 18 of `Advanced_Lane_lines.ipynb`.  

If I were going to pursue this project futher, I would try innovative approaches such as machine learning to find the lanes.
