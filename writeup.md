# **Finding Lane Lines on the Road** 

This file contains a written summary of the work performed to develop a processing pipeline for finding lane lines on the road.

## Pipeline description

The main function implementing the processing pipeline is called `process_image`. Please note that the function is used for **both image and video** processing steps. The only difference between image and video processing steps happen in the helper function `hough_lines`, as described later in section 5. 

The function `process_image` implements several steps, each of which is defined with a helper function. Some of the helper functions are the same as in the original assignment repository, without any modifications. Some of the helper functions have been modified, to achieve the assignment goals. The following sections describe the 6 steps in the pipeline and the corresponding helper functions. 

##### 1. Gray scale

To begin with, the input image is converted to grayscale, which is a standard step in edge detection process (it simplifies edge detection compared to doing it on the color images). The helper function for this step is not changed from the original content. 

##### 2. Blur

The second step in the pipeline is applying Gaussian blur to the image, to remove the noise and make the image smoother. This is done to help the Canny edge detection algorithm (in the next step) to ignore small, noisy and irrelevant edges. The helper function performing this step is not changed from the original content, and the kernel size used in the pipeline equals 5 (an empiracally reasonable value). 

##### 3. Canny edge detection

The third step in the pipeline performs Canny edge detection. This is one of the main functions in detecting the lane lines. The low and high threshold for the algorithm are specified to be 50 and 150, respectively. The helper function is not changed from the original content, and the image returned contains the edges highlighted in the image, while the rest is black. 

##### 4. Region of interest

In the fourth step, we define the region of interest by declaring the variable `vertices`, which contains vertices of a rectangle that encloses the area in the image that we want to process, while ignoring the rest. This approach works because we know where to expect lane lines if the camera is mounted at the frond end of the car and its position is always static. The vertices are derived approximately based on the images and videos resolution, and the helper function doing the work is not changed from the original content. The resulting image from this step contains Canny edges in the area enclosed by the rectangle, while the rest is simply black. 

##### 5. Hough lines

In the fifth step, we perform bulk of the work to accomplish the goal, which is to have a single line on each side of the field of view (left and right). We first find Hough lines using the OpenCV function in a standard way and providing the input parameters that work well for the given images and videos. This results in a certain number of lines on each side (typically double digit number of lines), which we want to combine into one line on the left side and one on the right. 

The lines are combined using the method `combine_lines`, which takes as inputs all the Hough lines detected and the vertices of the region of interest. This method uses `vertices` to understand which lines are on the left and which are on the right. The idea is to use the horizontal mid point of the vertices and assume that the lines with points left of the mid points are on the left, and the others are on the right. The alternative method would be to assign the lines to left or right by looking at the slope sign, but this approach was not used because sometimes the lines are found with slope equal to zero. 

The function `combine lines` has two main steps. 
- In the first step, `combine lines` uses the method `get_left_and_right_lines_bounds`, which assignes Hough lines to left and right sides, and for each side it calculates the average slope and intercept using **all** Hough line segments. From these average parameters, the method also calculates slope and intercept bounds (15-20% away from the average). Each Hough line segment needs to have slope and intercept within these bounds to be used in the second step. The lines with parameters outside of these bounds are considered outliers and ignored in the second step. The reasoning behind this approach is that we will get more accurate average values if we ignore the outliers.
- In the second step, `combine lines` uses the method `get_average_lines` to finally calculates the average slope and intercept for both sides. This time only the Hough line segments within the precalculated bounds are taken into account. 

Finally, `combine_lines` returns average lines on both sides, and each of them is specified as a separate list containint coordinates of two points. If a line on one side was not successfully detected or calculated, the corresponding list will be empty. 

At this point, we come to the only difference between the processing pipeline for images and the processing pipeline for the videos. 

- **For static images** we simply take the output of `combine_lines` and draw these lines on the image placeholder `line_img`
- **For videos** we might see that between successive frames the single lines on each side have relatively large differences in slope and/or intercept, which makes the line jump a lot between the frames. This is unpleasant and does not look nice, so we might want to do something to improve this. The idea used in this pipeline is to remember the slope and intercept from several previous frames (e.g. 5) and we calculate average of those historical parameters to get a bit smoother changes between frames. For this purpose, the `hough_lines` method uses variables `left_past` and `right_past`. For static images, they are set to `-1` and no averaging over successive frames happens (we are dealing with static single pictures). For videos, these variables are implemented as `deque`, and they always contain slopes and intercept of the last 5 (or another specified number) frames, including the current frame. So when the method `combine_lines` returns slopes and intercepts for the current frame, they are combined with the values from the previous frames to calculate the final averaged values. These values are then drawn in the image.  

The method `hough_lines` returns the image that contains up to one line for both left and right side of the field of view, and nothing else. 

##### 6. Draw the lines on top of the original image

In the final step, we take the image containing the lines from the previous step, and we draw the lines on top of the original image by using a standard OpenCV function. The helper function for this step has not been changed from the original content. The final outcome of this step is the image containing both the original content and the lines. This image can be saved, displayed, etc. 

## Areas for improvement

It is important to note that the pipeline in its current state works well on all of the images and videos provided with the project assignment, apart from the video in the optional extra challenge section. I wanted to submit the project before tackling this optional challenge, but I will try to continue working on it to develop the pipeline for that challenge as well. 
