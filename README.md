# Advanced Lane Finding

## Project Goals

The goals / steps of this project were the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/chessboard_3.jpg "Identified chessboard"
[image2]: ./output_images/chess_before.jpg "Chessboard before undistortion"
[image3]: ./output_images/chess_after.jpg "Chessboard after undistortion"
[image4]: ./output_images/persp_before.jpg "Identified road plane"
[image5]: ./output_images/persp_after.jpg "After perspective transform"
[image6]: ./output_images/edge_before5.jpg "Before edge detection A"
[image7]: ./output_images/edge_after5.jpg "Before edge detection A"
[image8]: ./output_images/edge_before1.jpg "Before edge detection B"
[image9]: ./output_images/edge_after1.jpg "Before edge detection B"
[image10]: ./output_images/pixel_windows.png "Identified lane pixels windows"
[image11]: ./output_images/fitted_from_windows.png "Lane lines fitted from window pixels"
[image12]: ./output_images/fitted_from_prior_poly.png "Lane lines fitted from prior poly"
[image13]: ./output_images/lane_after0.jpg "Final result A"
[image14]: ./output_images/lane_after1.jpg "Final result B"
[image15]: ./output_images/lane_after5.jpg "Final result C"
[image16]: ./output_images/undistort_before.jpg "Original image"
[image17]: ./output_images/undistort_after.jpg "Undistorted"

[video1]: ./output_images/out_project_video.mp4 "Video"

---

## Writeup

### Camera Calibration
In order to correct for camera distortions, photos with known reference points in them is needed. In this project, a chessboard pattern was used where all corners exist in the same plane with equal distances between them. OpenCV has functions for detecting the pixel coordinates (imgpoints) of chessboard corners with the `cv2.findChessboardCorners` function. The identified corners can also be visualized for verification with the `cv2.drawChessboardCorners` function, see results below.

![alt text][image1]

The chessboard corners are assumed to be fixed in the x,y - plane in world coordinates. So after identifying the image coordinates from a number of photos, the distortion coefficients and camera matrix can be estimated with the `cv2.calibrateCamera` function. With them the `cv2.undistort` function can be used to undistort images taken by this camera. This has been done to the chessboard image below

Before | After
- | -
![alt text][image2] | ![alt text][image3]

In the after image, the lines are now straight as expected.

## Pipeline (single images)

### 1. Distortion correction
The first step of the image pipeline is to use the estimated distortion coefficients and camera matrix. This is important in order to be able to get accurate real world distance measurements and a correct line curvature.

Before | After
- | -
![alt text][image16] | ![alt text][image17]

The code can be found in the notebook under headers *Find Chessboard Corners in Calibration Images* and *Calibrate Camera*

### 2. Perspective Transformation
The second step of my pipeline is transforming the perspective to a top down view. This was done by finding a polygon in the road plane of an image of a straight road and then finding the transformation matrix to a rectangle with the `cv2.getPerspectiveTransform` function. This matrix was then applied with the `cv2.warpPerspective` function. The manually adjusted source and destination polygon vertices are defined in the table below

| Source        | Destination   |
|:-------------:|:-------------:|
| 185, 720      | 300, 720      |
| 594, 450      | 300, 0        |
| 684, 450      | 980, 0        |
| 1100, 720     | 980, 720      |

The original and transformed images with the polygons visualized is shown in the figures below

Before | After
- | -
![alt text][image4] | ![alt text][image5]

The source polygon was adjusted until the lines in the transformed image were straight.

The code can be found in the notebook under header *Perspective Transform*


### 3. Edge Masking
I played around with a few different ways of masking out lane lines from the image.
For detecting the white lines, a channel threshold of the lightness (above 220) in a LUV representation was used. I also used a hue threshold for yellow which was fairly difficult to make work with the light colored asphalt. One of the better ways I found of detecting the yellow lines were with a threshold on the B-channel of the LAB color space. I used a threshold of [150, 220] for this.

Another strategy I used was to first calculate an x-derivative with a Sobel operator and then attempting to find sections of the gradient image where a large positive derivative is followed by a large negative one. The positive derivative comes from going from asphalt to lane line, and the negative from going back to the asphalt again. This is a characteristic which larger shadows and center barriers don't replicate to a high extent so it worked as a strategy to exclude those edges. I used this approach both on a lightness channel Sobel mask and a blue channel Sobel mask from the RGB color space.

The final mask I used was to include all different approaches at once (logical OR). I have visualized the lightness Sobel mask, B-channel threshold mask and lightness threshold mask as red green and blue respectively in the figures below

Before | After
- | -
![alt text][image6] | ![alt text][image7]
![alt text][image8] | ![alt text][image9]

The shadows together with the desaturated and distorted lines make the second quite difficult to find.

The code can be found in the notebook under header *Edge Detection*


### 4. Lane-line Pixel Identification and Polynomial Fit
In order to make an accurate polynomial fit to the lane line pixels, two different lane line pixel finding algorithms were used. The first approach which does not require any previous information is a method for finding windows with a large number of pixels in them starting from the bottom of the image and only searching fairly close to the previous window. The maximum was calculated with a box convolution and the results can be seen in the figure below

![alt text][image10]

The white pixels within the green windows are kept while the pixels outside are rejected.

The other approach is to use a previously fitted polynomial and select all pixels within a band around it. 

In either case, the selected pixels were used to fit a second order polynomial. The resulting lines can be seen in the figures below

From Windows | From Priors
- | -
![alt text][image11] | ![alt text][image12]

The code can be found in the notebook under header *Find Lane Pixels and Estimate Polynomial*


### 5. Radius of Curvature and Lateratl Position Calculation
In order to estimate the lane curvature and the lateral position of the car, the lane line polynomials were estimated after rescaling the lane line pixel coordinates to real world distances using the fixed factors 3.7 / 660 for the x-axis and 30 / 60 for the y-axis. These fractions were based on the assumed length of dashed lines and width of a lane.

The curvature was then estimated with the equation
`((1 + (2*fit[0]*y_eval*ym_per_pix + fit[1])**2)**1.5) / np.absolute(2*fit[0])`
where y_eval was chosen to the point of halfway between the car and the horizon. I also estimated the turn direction based on the coefficients in the polynomial fit. If the first coefficient is negative and the second is positive the lane is turning to the left. If the signs of the coefficients are the opposite it is turning to the right. If the curvature of left and right lines are in different directions or the radius is below 100 m the curvature is set to "unknown". If the curve radius is above 4 km I consider it a straight line.

In order to estimate the lateral lane position, the camera is assumed to be centered in the vehicle. Then the lower points of the real world scaled lane line polynomials are used to determine the distance of their center point from the image center point.

The code can be found in the notebook under header *Estimate Curve Radius and Lateral Position*


### 6. Lane Visualization and End Result
The end result is visualized by rendering a green polygon between the fitted lane line polynomials in an image which is than inverse transformed from the top-down view back to the original view where its is overlayed on the original image. The current curve type and radius estimations as well as the lateral position is rendered with text in the top left corner. The end result can be seen in three sample images below.

![alt text][image13]
![alt text][image14]
![alt text][image15]

The code can be found in the notebook under header *Draw Lane Overlay and Information*

---

## Pipeline (video)

### 7. Temporal Filtering
In order to use more information than a single frame in the video, I keep the previous 8 lane line estimations as state and take an unweighted average of the coefficients of the polynomials. I also do some simple outlier rejection by discarding frames which I consid having "unknown" turn direction. If a sufficiently large number of frames in the state agree on the turn direciton, only those frames will be used in the average. For example if 6 of 8 frames consider it a left turn, 1 frame is unknown and 1 frame is a right turn. Only the 6 left turn frames will be averaged together for the final estimate.

The code can be found in the notebook under header *Time Filtering*


### Result
The final result can be seen in this video: [link to video result](video1)

---

## Discussion
The lane finding pipeline implemented was quite sensitive to different lighting conditions and lighter colored asphalt with faded lane lines. These problems were apperent already in the edge masking step of the pipeline which means the biggest improvements should come from improving that step or the ones before it.

One limitation which comes from the 8 frame filter is handling rapid pitch variations of the car when riding over bumps. The filter is too slow to adapt to these changes which makes the end result look slightly wonky. The varying pitch is also a problem for the hard coded perspective transformation.

The second order polynomial fit also showed some problems where the a bent curve would be fitted to a straight dashed line in some cases. Investigating other approaches for this could be another way forward, perhaps using information from both lines at the same time since they should be similar in most cases.

Other limitations include, but are not limited to, handling situations involving other cars, lane changes, construction work, different weather, night time etc.
