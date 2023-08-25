# Advanced Lane Finding

## Introduction
Welcome to the Advanced Lane Finding GitHub repository! 🛣️ Here, we've crafted a cutting-edge lane detection pipeline that combines perspective transformation, color analysis, and polynomial fitting to accurately identify lanes on the road. Whether you're a developer or an enthusiast, join us in exploring the world of advanced computer vision for safer and smarter driving! 🚗👀

[//]: # (Image References)

[image1]: https://github.com/yukitsuji/Advanced_Lane_Finding/blob/master/output_images/calibration1.jpg?raw=true "Undistorted"
[image2]: https://github.com/yukitsuji/Advanced_Lane_Finding/raw/master/output_images/calibration2.jpg "Road Transformed"
[image3]: https://github.com/yukitsuji/Advanced_Lane_Finding/raw/master/output_images/bird_view.jpg "Bird View Image"
[image4]: https://github.com/yukitsuji/Advanced_Lane_Finding/raw/master/output_images/thresholding.jpg "Thresholding"
[image5]: https://github.com/yukitsuji/Advanced_Lane_Finding/raw/master/output_images/histogram_filtering.jpg "Fit Visual"

### Camera Calibration

#### 1. Computing the camera matrix and distortion coefficients.

The code for this step is contained in the first code cell of the IPython notebook located in `./Advanced_Lane_Finding.ipynb` (or in lines 12 through 67 of the file called `main.py`).  

```python
def get_calibration_param(image_url):
    images = glob.glob(image_url)

    objp = np.zeros((6 * 9, 3), np.float32)
    objp[:, :2] = np.mgrid[0:9, 0:6].T.reshape(-1, 2)

    objpoints = []  # 3d points in real world space
    imgpoints = []  # 2d points in image plane.
    corner = (9, 6)

    for image in images:
        img = mpimg.imread(image)
        gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
        ret, corners = cv2.findChessboardCorners(gray, corner, None)

        if ret:
            objpoints.append(objp)
            imgpoints.append(corners)

    img_size = (img.shape[1], img.shape[0])
    ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(objpoints, imgpoints, img_size, None, None)
    print("finish get_calibration_param")
    return mtx, dist


def get_undistortion(distorted_img, mtx, dist):
    undist = cv2.undistort(distorted_img, mtx, dist, None, mtx)
    return undist


def input_calibration_file(path="./calibration.npz"):
    """get camera matrix and distortion coefficients from file"""
    try:
        calibration_param = np.load(path)
        return calibration_param['mtx'], calibration_param['dist']
    except IOError as e:
        print(e)
        raise IOError("Please Set Correct Calibration File")
```

![alt text][image1]

### 1. Example of a distortion-corrected image.
To demonstrate this step, We will describe how We apply the distortion correction to one of the test images like this one:

![alt text][image2]


### 2. Performing a perspective transform.

We chose the hardcode the source and destination points in the following manner:

```python
src = np.float32([[490, 482],[810, 482],
                  [1250, 720],[40, 720]])

dst = np.float32([[0, 0], [1280, 0],
                 [1250, 720],[40, 720]])
```
This resulted in the following source and destination points:
```
|  Source   | Destination |
|:---------:|:-----------:|
| 490, 482  |    0, 0     |
| 810, 482  |   1280, 0   |
| 1250, 720 |  1250, 720  |
|  40, 720  |   40, 720   |
```

![alt text][image3]

### 3. Using color transforms, gradients to create a threshold binary image.

Here's an example of our output for this step.

![alt text][image4]

### 4. Identifying lane-line pixels and fit their positions with a polynomial.
- Identify peaks in a histogram of the image to determine location of lane lines.  
- mask images by peaks we get.
- Fitting a polynomial to each lane using the `cal_poly` method in lines 273 in `main.py`.

```python
def cal_poly(img, left_boundary, right_boundary):
    side_img = img[:, left_boundary: right_boundary]
    index = np.where(side_img == 1)
    yvals = index[0]
    xvals = index[1] + left_boundary
    if xvals.size != 0:
        fit_equation = np.polyfit(yvals, xvals, 2)
        yvals = np.arange(img.shape[0])
        fit_line = fit_equation[0] * yvals ** 2 + fit_equation[1] * yvals + fit_equation[2]
        return fit_line, fit_equation
    else:
        return 0, np.array([10000., 100., 100.])
```

Once we've found lane lines in one frame of video, and they are actually the lines we are looking for, We can simply search within a window around the previous detection from next frame of video.  
- Search for the new line within +- 30 pixel around old line center.

For determining if detected lines are the real things, we check lines
- Checking that left and right lines are roughly parallel.
- Cheking that the difference between previous and current lines curvature are little. 


![alt text][image5]

### 5. Calculating the radius of curvature of the lane and the position of the vehicle with respect to center.

- We calculate curvature by using `__cal_curvature` method in lines 312 through 322 in `main.py`.  

```python    
        def __cal_curvature(self):
        """Compute radius of curvature for each lane in meters"""
        ym_per_pix = 30. / 720
        xm_per_pix = 3.7 / 700

        left_fit_cr = np.polyfit(self.yvals * ym_per_pix, self.left_line.bestx * xm_per_pix, 2)
        right_fit_cr = np.polyfit(self.yvals * ym_per_pix, self.right_line.bestx * xm_per_pix, 2)

        self.left_line.radius_of_curvature = ((1 + (
                    2 * left_fit_cr[0] * np.max(self.yvals) + left_fit_cr[1]) ** 2) ** 1.5) \
                                             / np.absolute(2 * left_fit_cr[0])
        self.right_line.radius_of_curvature = ((1 + (
                    2 * right_fit_cr[0] * np.max(self.yvals) + right_fit_cr[1]) ** 2) ** 1.5) \
                                              / np.absolute(2 * right_fit_cr[0])
```

Calculate the average of the x intercepts from each of the two polynomials position = *(rightx_int+leftx_int)/2*

- We calculate the position of the vehicle by using `add_place_to_image` method in lines 325 through 335 in `main.py`.

```python
    def add_place_to_image(self, image):
        place = (self.left_line.bestx[-1] + self.right_line.bestx[-1]) / 2
        diff_center = np.abs((image.shape[1] / 2 - place) * 3.7 / 700)

        if place > image.shape[1] / 2:
            cv2.putText(image, 'Vehicle is {:.2f}m left of center'.format(diff_center), (100, 80),
                        fontFace=16, fontScale=2, color=(255, 255, 255), thickness=2)
        else:
            cv2.putText(image, 'Vehicle is {:.2f}m right of center'.format(diff_center), (100, 80),
                        fontFace=16, fontScale=2, color=(255, 255, 255), thickness=2)
        return image
```

Calculated the distance from center by taking the absolute value of the vehicle position minus the halfway point along the horizontal axis distance_from_center = *abs(image_width/2 - position)*

**Note**  
The distance from center was converted from pixels to meters by multiplying the number of pixels by 3.7/700.

### 6. Providing the computed output

We implemented all pipeline in `main.py`. Main method is `process_image` in line 363 in `Line_detector` class.

## 🤝 Contributors
- @(Vaibhav Mishra)[https://vaibhav-mishra.vercel.app/]
- @(Gaytri Sran)[https://gaytrisran03.github.io/portfolio-gaytri/]

 Thank You
---
