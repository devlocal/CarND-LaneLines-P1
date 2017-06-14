# **Finding Lane Lines on the Road** 

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./filter_illustrations/whiteCarLaneSwitch-intersect-off.jpg "filter_by_intersection_with_bottom_edge off"
[image2]: ./filter_illustrations/whiteCarLaneSwitch-intersect-on.jpg "filter_by_intersection_with_bottom_edge on"
[image3]: ./filter_illustrations/filter_by_endpoints-off.png "filter_by_endpoints off"
[image4]: ./filter_illustrations/filter_by_endpoints-on.png "filter_by_endpoints on"
[image5]: ./filter_illustrations/filter_by_slope-off.png "filter_by_slope off"
[image6]: ./filter_illustrations/filter_by_slope-on.png "filter_by_slope on"
[image7]: ./filter_illustrations/merge_segments-off.png "merge_segments off"
[image8]: ./filter_illustrations/merge_segments-on.png "merge_segments on"

---

## Reflection

The project contains three pipeline setups. The first is a simple pipeline that detects line segments, the second is an improved pipeline that extrapolates line segments, and the third is a pipeline that attempts to tackle the challenge.

The pipelines use helper functions provided in the pipeline template as well as new functions introduced by the student.

### Simple pipeline

A simple pipeline consists of the following steps:
1. The image is converted to grayscale;
1. Gaussian blur is applied with kernel size 5;
1. To find pixels with high gradient, Canny edge detection is applied with *low threshold* 50 and *high threshold* 150;
1. A mask is applied to the output of Canny edge detection to exclude lines not in the region of interest. A region of interest is a trapezoid with bottom base going horizontally at the bottom of the image from 5% to 95% of image width and top base going horizontally at 60% of image height from 45% to 55% of the image width.
1. Hough transformation is applied to edge image to find line segments with parameters rho=2, theta=(np.pi/180), threshold=15, min_line_len=40, max_line_gap=20;

The function `test_pipeline` runs the simple pipeline. For each image, it creates a 'region of interest' mask, then invokes `mark_lines` to annotate the image. The image is then plotted in the notebook and saved on disk.

The function uses `figsize` and `DPI` parameters of `plt.figure` to draw images at 100% scale in order to better show details, otherwise matplotlib outputs images with too small of a scale.

The function `draw_lines` is eventually called by `hough_lines` function to draw line segments.
>Note: The `draw_lines` function has additional capabilities used by other setups of a pipeline. Those capabilities are disable in the simple pipeline by `SIMPLE_PARAMS` configuration dictionary.

The function `get_lane_lines` is called by `draw_lines` to find lane lines in the image. `get_lane_lines` calls `searate_by_slope` to separate lines into two groups of *left lines* and *right lines*.

The function `get_lane_lines` calls `filter_by_intersection_with_bottom_edge`: a simple filter that finds intersection point of a line with the bottom edge of the image and filters out line segments that intersect the bottom edge beyond region of interest. This step helps to remove some noise (i.e. line segments that do not represent lane lines) from the list of lane line candidates. The assumption behind this filter is that the lane is centered in the frame and the camera has wide lens to capture larger area than the lane with its lines.

### Improved pipeline

Improved pipeline performs all the steps of a simple pipeline with some additional steps. Improved pipeline is configured by `IMPROVED_PARAMS` dictionary which has `extrapolate` and `filter_by_endpoints` set to `True`. The `extrapolate` setting enables drawing extrapolated lane lines by the `draw_lines` function, while `filter_by_endpoints` adds an additional step to filter line segments.

`get_lane_lines` calls `filter_by_max_x` and `filter_by_min_x`: a simple filter which looks at the coordinates of end points of each line and removes left or right lines that go too far (more than 60%) on the other half of the image. The assumption behind this filter is that lane lines are mostly centered which holds true for the test media (images and video clips) provided.


`draw_lines` function calls `draw_extrapolated` for left and right lines separately.

`draw_extrapolated` first extrapolates line segments by calling `extrapolate_to` and then draws extrapolated line.

The function `extrapolate_to` calls `average_line_fn_params` to average detected line segments.
>Note: the function also contains code for the merging step which is skipped in the improved pipeline.

`average_line_fn_params` calls `line_segment_to_fn` to find parameters `k` and `b` of each line segment for line formulation *y=k\*x+b*. The parameters are then averaged and an extrapolated line segment coordinates are computed.

### Challenge pipeline

The last pipeline attempts to tackle the challenge video. It has all the steps in the improved pipeline plus some additional steps. Its configuration is defined by `CHALLENGE_PARAMS` dictionary which has `filter_by_slope` and `merge_segments` set to `True`.

`filter_by_slope` enables one more filtering step in the `get_lane_lines` which removes lines that are "too horizontal", i.e. lines that have too large value of slope `k` for x=k\*y+b line formulation.

Setting `merge_segments` to `True` enables `merge_line_segments` and `get_longest` function calls in the `extrapolate_to` function. The call of `merge_segments` merges individual line segments that lay on a single geometrical line, and the call of `get_longest` finds the longest line in the list of line candidates. These two calls allow to find the group of lines that dominate in the half-image (either left or right part).

## Filter illustrations

This section illustrates how different algorithmic filters applied by a pipeline change lane lines detection result.

### `filter_by_intersection`

Notice unwanted line segment near the far end of the left line when the filter is off.

Filter off:
![alt text][image1]

Filter on:
![alt text][image2]

### `filter_by_endpoints`

Notice unwanted line segment in the bottom right part of the first image (marked with blue) and how it affects averaged line (marked with red).

Filter off:
![alt text][image3]

Filter on:
![alt text][image4]

### `filter_by_slope`

Notice how an edge detected on the hood of the car affects averaged line.

Filter off:
![alt text][image5]

Filter on:
![alt text][image6]

In the challenge pipeline, this filter has little effect on the quality of lane lines detection because when `merge_segments` filter is enabled the pipeline takes the longest merged segment and in majority of frames insignificant 'noise` segments with extreme slope are ignored.

### `merge_segments`

Notice how unwanted segments in the shade on the left displace the left line.

The filter merges line segments and takes the longest segment ignoring all shorter segments, therefore the second image contains the same line segments in the shade area, however they are not used for final line averaging and extrapolation.

Filter off:
![alt text][image7]

Filter on:
![alt text][image8]

## 2. Identify potential shortcomings with your current pipeline

1. Filter functions `filter_by_max_x` and `filter_by_min_x` work based on the assumption that lane lines are positioned at certain regions of an image. This assumption, although being acceptable for the test media files, may break for various real-life scenarios of roads with high curvature, turns or when a vehicle changes lanes.
1. Similarly, `filter_by_intersection_with_bottom_edge` assumes that the vehicle drives parallel to lane lines, and that the lane lines are straight or slightly curved, which may not always be true in real life situations.
1. The model of a lane line assumes that a lane line (either solid or dashed) goes straight which is not always true.
1. Lane lines detected on the `solidYellowLeft.mp4` video clip are 'shaky' frame to frame.


## 3. Suggest possible improvements to your pipeline

1. Parameters of Canny edge detection and Hough transform can be better tuned with some more experimentation;
1. Challenge pipeline can pre-process image to let Canny edge detection work better, for example by applying yellow filter first.
An assumption behind this improvement is that yellow filter would make both yellow and white lines look similar on grayscale image which would allow to tune Canny edge detection to get more accurate results.
1. Since a lane line on a road has thickness, there expected be two edges of a line going parallel. A pipeline could use this fact to better distinguish between lane lines and noise.
1. As an advancement of the previous improvement, a pipeline could check the shape of detected lane line segments (lane line segments have known shape, typically long and thin).
1. A more advanced lane line model could represent a lane line as an arc, rather than a straight line.
1. A frame-to-frame stabilization of detected lane lines could be applied to remove shakiness of detected lines.
