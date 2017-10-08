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

[image0]: ./test_images/test7.jpg "Orig image"
[image1]: ./output_images/undistorted_images/test7.jpg "Undistorted"
[image2]: ./output_images/binary_images/test7.jpg "Binary image"
[image3]: ./output_images/binary_images_lines/test7.jpg "Binary image lines"
[image4]: ./output_images/perspective_transform_lines/test7.jpg "Perspective transform lines"
[image5]: ./output_images/fit_lane/test7.jpg "Fit lanes"
[image6]: ./output_images/final_image/test7.jpg "Output"
[image7]: ./radius_of_curvature.jpg "Radius of curvature"
[image8]: ./polynomial.jpg "Polynomial"

## [Rubric Points](https://review.udacity.com/#!/rubrics/571/view)

Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in cells 2-4 of `Advanced Lane Lines.ipynb` Jupyter notebook .  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image

![alt text][image0]

using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

The distortion correction transform stretches the image. For example,  the rear left light of the white car disappears after this transformation. Additional examples of undistorted images are located in output_images/undistorted_images folder.

### Pipeline (single images)

The pipeline for test images is managed by the code in cell 13 of `Advanced Lane Lines.ipynb` notebook.

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I describe how I apply line detection pipeline to one of the distortion-corrected test images like this one:
![alt text][image1]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (`create_binary_image()` functions in cell 5, this code is called from cell 6 of `Advanced Lane Lines.ipynb`). I transformed an image into HLS space and computed Sobel operator along x axis in L channel. The output of Sobel operator war normalized to be in the range of [0,255]. In the thesholded binary image a pixel has white color if its Sobel operator is in the range [20,100] or both its S channel value is in [100,255] range and its L channel value is in [50,255] range. Here is an example of my output for this step:

![alt text][image2]

Additional examples of thresholded binary images are located in output_images/binary_images folder.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is in cells 7 and 8 of `Advanced Lane Lines.ipynb` notebook. I chose the following values of the source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 609, 439      | 300, 0        |
| 669, 439      | 980, 0        |
| 1046, 673     | 980, 720      |
| 257, 673      | 300, 720      |

I used the function `cv2.getPerspectiveTransform()` to create a perspective transformation matrix. I also used this matrix and the function `cv2.warpPerspective()` to remove the effects of perspective view. I verified that my perspective transform was working as expected by drawing the polygon, created by `src` and `dst` points, onto a test image

![alt text][image3]

 and its warped counterpart to verify that the polygon and lane lines appear parallel in the warped image.

![alt text][image4]

Additional examples of images with perspective polygon before and after perspective transform are located in output_images/binary_images_lines and output_images/perspective_transform_lines folders respectfully.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I identified left and right lane lines in the gray scale image using a sliding window technique:

1. **Find base points of left and right lines**.  I computed a histogram of white pixels in the bottom strip of the image. This strip has height 200 and the same width as the gray scale image. In this histogram I counted the number of white pixels that have the same x value. Then I split the histogram into the left and right parts. The x value with the largest histogram value in the left part is the base of the left line. Similarly, the x value with the largest histogram value in the right part is the base of the right line. The code for this step is in the function `find_lane_base()` in cell 11.

2. **Initialize a search window**. I split y axis into 20 equal search windows of length image height/20. I created two rectangles, each one centered at the base point of left/right line. The height of each rectangle equals to the length of the search window. The rectangle has width 260. Initially rectangles are placed in the bottom search window.

3. **Track a lane line by sliding a search window**. In the beginning of tracking process the rectangles are placed in the bottom search window. All white pixels that are inside left/right polygon are assigned to the left/right line. If the number of pixels in rectangle is larger than 10 then in the next window the rectangle is re-centered to the mean x values of its white points in the current window. Then the rectangle is moved up to the next search windows and the pixel assignment process continues. The image below shows that when there is a significant curvature, the white pixels of different lines merge in the top of the image. This makes very hard to track lines in the top part of the image. For this reason I did not use the top 5 search windows.

  After assigning white pixels to lines, I use `np.polyfit()` function to find a second-degree polynomial that represents a line. Since frequently the lane lines are nearly vertical, the fit is done with x coordinates being a function of y coordinates.   

The code for steps 2 and 3 is in the function `track_lane()` in cell 11. The functions `find_lane_base()` and `track_lane()` are called from the function `find_line()` (cell 12), which is the main function for finding and visualizing lanes.

The following image shows the rectangles as they move through search windows, pixels assigned to each line and corresponding polynomials. Red pixels are assigned to the left line and red pixels are assigned to the right line.

![alt text][image5]

Additional examples of such images are located in the folder output_images/fit_lane folder.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The computation of the radius of curvature is performed in function `radius_of_curvature()` in cell 11 and consists of the three steps:

1. Convert the pixels assigned to lane line from image space to real world space. I assumed that 30 meters along y axis correspond to 720 pixels and 3.7 meters along x axis correspond to 700 pixels. Using this assumption, to transform coordinates from image to pixel space we multiply x and y values of pixels by 3.7/700 and 30/720 respectfully.

2. Fit a second degree polynomial over the line pixels using their real world coordinates. I used `np.polyfit()` function. Similarly to fitting polynomials in the image space, the fit in the real world space is done with x coordinates being a function of y coordinates. The resulting polynomial can be represented as

   ![alt text][image8]

3. Compute the radius of curvature at the base of the lane line using the formula:

   ![alt text][image7]  

   where A and B are coefficients of the second degree polynomial found in the previous step and y is an y coordinate of the line base.

The computation of the position of the vehicle with respect to the center is done in function `find_line()` in cell 12 and consists of the following two steps:

1. Let x_left and x_right be the x coordinates of the bases of the left and the right lines. I estimated center of the lane to be (x_left + x_right)/2.

2. Since the camera is centered and points straight ahead, car center always has x coordinate (image width)/2. If the car drives in the center of the lane then the estimated center of the lane is equal to (image width)/2. In the general case the distance in image space between the estimated center of the lane and the car center is |(x_left + x_right)/2 - (image width)/2|. If (x_left + x_right)/2 - (image width)/2 is larger than zero then the car is to the left of the lane center, otherwise the car is to the right of the center. To convert the distance from image space to real world space, I multiplied |(x_left + x_right)/2 - (image width)/2| by 3.7/700, similarly to the conversion performed when computing radius of curvature.    

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the function `visualize_lanes()` in cell 12. I used the lane lines to create a polygon that represents the lane area. This polygon is drawn over the black image and is filled with the green color using the function `cv2.fillPoly()`. The resulting visualization of the drivable area is in the image space after perspective transform. In the next step I converted this polygon to the original image space using inverse perspective transform. The transformation matrix of this transform was obtained previously along with the matrix of perspective transform. Finally, I merged the original undistorted image with the image of the polygon in the original image space. Here is an example of my result on a test image:

![alt text][image6]

Additional examples of such images are located in the folder output_images/final_images.

---

### Pipeline (video)

The pipeline for processing video frames is managed by function `process_image_continuous()` in cell 15 of `Advanced Lane Lines.ipynb` notebook.
In addition to the steps used to process a single image, I used two techniques to speed up the computation and improve the accuracy of finding the drivable area:

1. **Adjustment of lane lines from the previous frame**. After finding lane lines in the current frame I save them  for a possible reuse in the next frame. When finding lane lines in a new video frame, before starting the sliding window process (step 4 above) I look at the white pixels that are within margin of 50 pixels from the lane line of the previous frame. Then I fit a second degree polynomial using these pixels. I repeat this step for both left and right lines. If the new lines are convex or concave then I take these lines and skip the sliding window process. Otherwise, I find lane lines from scratch using sliding window process.

2. **Smoothing of lane lines**. The final line in the current frame is the weighted average of the lines found in the current and previous frames.
After several experiments I found that 0.5 weight of current and previous frames gives the most accurate line detection.

Here is a [link to my video result](./output_videos/project_video.mp4).

---

### Discussion
In this project I showed how to detect lane lines, identify lane area, compute radius of curvature and position of a car within a lane. There are several points for future improvements and enhancements:  
1. Make the pipeline work well on two other videos (`challenge_video.mp4` and `harder_challenge_video.mp4`).
2. The test video and images have a car diving in the leftmost or rightmost lane. The pipeline should also be tested when the car drives in a central lane.
3. The pipeline can be enhanced to handle roads with only a single lane line.
4. Handling of splitting and merging lanes.
5. Handling of roads that have pedestrian crossing markings.
6. Lane line detection pipeline can be merged into behavior cloning model from the previous project. The output of lane lines pipeline can be an additional input to a neural network that predicts steering angle.  
