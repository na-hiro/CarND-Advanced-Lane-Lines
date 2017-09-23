## Writeup
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

[image1]: ./output_images/01_all_checkboad.png "Checkboads"
[image2]: ./output_images/02_undistort_test.png "undistort test"
[image3]: ./output_images/03_undistorted_sample.png "undistort sample"
[image4]: ./output_images/04_warped.png "warped"
[image5]: ./output_images/05_hls_img.png "Checkboads"
[image6]: ./output_images/06_color_th.png "Checkboads"
[image7]: ./output_images/07_combined.png "Checkboads"
[image8]: ./output_images/08_pipeline.png "Checkboads"
[image9]: ./output_images/09_slide_window.png "Checkboads"
[image10]: ./output_images/10_histgram.png "Checkboads"
[image11]: ./output_images/11_polyfit_prev.png "Checkboads"
[image12]: ./output_images/12_measure_lane.png "Checkboads"
[image13]: ./output_images/13_draw_pos.png "Checkboads"
[image14]: ./examples/color_fit_lines.jpg "Checkboads"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

The code for this step is contained in "./Camera_Calibration.ipynb".  

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function and obtained this result:

![alt text][image1]

I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image2]

### Pipeline (single images)
The code for this step is contained in "./CarND-Advanced-Lane-Lines.ipynb".  

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![alt text][image3]

#### 2. Input image format
The selection of the input image for the main image processing is explained here.Firstly, I examined which format the input image of threshold processing should be. Is it a normal view image or a bird's-eye view image? In conclusion, I selected a bird's eye view as an input image. Since it is necessary for fitting to be done on a bird's-eye view, overlapping processing can be omitted. Secondly, the bird's eye view also has the effect that it is easier to intuitively understand information such as angle.

#### 3. Performed a perspective transform
Adopting a bird's-eye view as an input image for image processing was explained in the previous section.

My perspective transform includes a function called `exec_warp()`.The `exec_warpe()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  

The source src is shown in the lower left figure, and the result after the conversion is shown in the lower right figure. The lane of the src image is straight, so you can see that the converted image is kept almost parallel.

![alt text][image4]

#### 4. Threshold image processing
This section explains threshold processing for extracting the points that make up the lane.

##### 4.1 Color space conversion
In order to efficiently extract lanes with different colors, the image in RGB space is converted to the image in HLS color space space, as input data for thresholding.

The results of converting the RGB image into the HLS image are shown below.

![alt text][image5]

##### 4.1.1 Color Threshold Processing
The threshold processing is performed on the HLS image and the result of extracting the lane candidate is shown below.
It was confirmed that yellow and white lanes were extracted efficiently. The function name of this operation is `det_lines_by_hls_color()`.

![alt text][image6]

##### 4.1.2 Threshold processing of gradient and its direction
Here, I explain the magnitude of the gradient of the HLS image and the thresholding of that direction. By combining this processing with color threshold processing, narrowing down of lane candidate points becomes possible.The function name of this operation is `combine_color_and_gradient()`.

![alt text][image7]

The result of this processing is shown in the lower left figure, and the result of combining with the color threshold processing is shown in the lower right figure.
In addition, the results applied to all test data are shown below. It was confirmed that the candidate points of the lane were extracted even in the image that the lane was difficult to confirm.

![alt text][image8]

#### 5.Curve approximation processing
In this section, a method to approximate extracted lane candidate points to quadratic curves is explained.

##### 5.1 Finding Sliding Window

Here, the points extracted as candidate points of the lane by thresholding are further selected. The selection method is as follows. A small window is created, and points extracted by thresholding in that window are selected as carefully selected points.

As for the window creation method, a histogram is generated with the image after thresholding as the axis in the horizontal direction, and a small window is generated around the peak point. Multiple windows are created on one line and are generated at the optimum position while being updated sequentially.

The generated sliding window is shown below.

![alt text][image9]

The histogram is shown below.

![alt text][image10]

##### 5.2 Processing using past detection results
The screening process by window generation described in the previous process does not necessarily have to be performed every frame. If a normal line is detected in the previous frame, it is efficient to use past lane information as window and center line. Furthermore, by keeping each line as a class, it is possible to detect robustly by holding several frames of information, optimum fit result, etc. and using it.

The lane candidate points carefully selected by the above processing are approximated to a quadratic curve and the average of several(max.:10) frames is judged to be optimum.
The extraction processing results using past frames are shown below. The green area is generated from past detection results.

![alt text][image11]

#### 6.Measuring Curvature
This section describes how to calculate curvature and car position.
First of all, we measure the relationship between the distance of the real space and the image converted to the bird's-eye view. The correspondence is shown in the figure below. From now on, calculate the distance (transformation coefficient) of the real space per pixel and calculate the curvature.The function name of this operation is `calc_curv_rad_and_center_dist()`.

![alt text][image12]

Furthermore, the position of the car is calculated by multiplying the center deviation of the left and right lanes by the conversion factor.The reference diagram is shown below.The quadratic approximate expression on the bird's eye view is as shown in the figure below.

![alt text][image14]

#### 7.Drawing a lane

This section explains how to draw the detected lane on the normal view image.

Detection of the lane in the bird's-eye view has been completed so far. After that, it just converts it to normal view image information. Perspective conversion was performed as an input image of threshold processing to generate a bird's-eye view. Drawing the lane is completed by alpha blending the result after inverse transformation of the lane drawn as a bird's-eye view and the image after the distortion correction processing.

The results of all the processing are shown below.

![alt text][image13]

With the processing so far, processing of one image has been completed. In the case of movies, this process is done in succession.


### Pipeline(video)
#### 1. Provide a link to my final video output.  

Here's a [link to my project video result](./project_video_output.mp4)

Here's a [link to my challenge video result](./challenge_video_output.mp4)

Here's a [link to my harder_challenge video result](./harder_challenge_video_output.mp4)

### Discussion

The proposed method was applied to evaluation videos of P4.
The results of project_video.mp 4 and challenge_video.mp 4 can be judged to be generally good. However, it was confirmed that the result of harder_challenge_video.mp 4 is not good. Improvement methods are as follows. harder_challenge_video.mp4 is a road in the mountains and it is quite a sharp curve. It can be judged that it is difficult to approximate by a quadratic curve. In such a case, narrowing the detection target range limits the fitting target to the vicinity. Also, as a condition of the lane, it is thought that the change of the lane width, the lane width from the neighborhood to the distance, can be used to exclude outliers. In addition, as the best fit, we believe that improvement in accuracy can be expected by introducing evaluation criteria such as scores instead of simple average.
