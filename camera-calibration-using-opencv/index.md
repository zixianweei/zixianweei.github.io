# <i class='fa-solid fa-language fa-lg'></i> OpenCV相机标定


{{< admonition type=info title="FYI" open=false >}}
原文来自[OpenCV官方文档](https://docs.opencv.org/4.8.0/dc/dbb/tutorial_py_calibration.html)，本人在翻译过程中进行了简单加工和信息补充。如果您认为侵犯了您的合法权益，请与我联系，我将删除本文。
{{< /admonition >}}

## 本文目标

通过本文，我们将会学习与如何使用OpenCV进行相机标定相关的知识。具体的：

1. 由相机导致的图像畸变有哪些类型；
2. 如何通过计算得到相机的内参矩阵和外参矩阵；
3. 如何利用相机的内参矩阵和外参矩阵对图像进行消畸变。

## 基础知识

基于小孔成像模型的相机会为所拍摄的图像引入图像畸变问题。其中，最主要的畸变类型是：**径向畸变**（radial distortion）和**切向畸变**（tangential distortion）。

径向畸变会导致真实场景中的直线变为图像中的曲线。该现象具有越远离图像中心越明显的特点。在图1中，棋盘格两侧使用了两条红色参考线作为对比，可以发现：棋盘格的边缘不再是一条直线；与红色参考线相比，弯曲的线条向图像边缘膨胀；且越靠近图像边缘，线条向外膨胀的现象越明显。如果想要了解更多有关图像畸变的知识，可以参考维基百科中的[**光学畸变**](https://en.wikipedia.org/wiki/Distortion_%28optics%29)一节。

<figure>
  <img src="calib_radial.jpg" style="width:50%;" alt="calibration radial">
  <figcaption>
    <h4>图1 径向畸变现象展示</h4>
  </figcaption>
</figure>

具体的，径向畸变可以使用下面的公式表示：

$$
\begin{align*}
x_{distorted} &= x(1 + k_{1} r^2 + k_{2} r^4 + k_{3} r^6), \\\\
y_{distorted} &= y(1 + k_{1} r^2 + k_{2} r^4 + k_{3} r^6).
\end{align*}
$$

与径向畸变类似，切向畸变则是由相机镜头所在平面和像平面非严格平行引起的。因此，图像中的部分区域会存在看上去比理论上更靠近的现象。切向畸变可以使用下面的公式表示：

$$
\begin{align*}
x_{distorted} &= x + [2p_{1}xy + p_{2}(r^2+2x^2)], \\\\
y_{distorted} &= y + [p_{1}(r^2+2y^2) + 2p_{2}xy].
\end{align*}
$$

根据径向畸变和切向畸变的表示公式，我们可以使用五个参数表示图像的畸变。这五个参数也被称为**畸变参数**（distortion coefficients）：

$$
\begin{align*}
C_{distortion} = (k_1 \enspace k_2 \enspace p_1 \enspace p_2 \enspace k_3).
\end{align*}
$$

除畸变参数外，我们还需要一些额外的信息来完成畸变矫正的流程，如相机的内部参数和外部参数。

每个相机的内部参数均不相同，他取决于相机的光学特性。通常情况下，相机的内部参数由相机的焦距 $(f_x, fy)$ 和光心位置 $(c_x, cy)$ 组成。由于每个相机的内部参数都是独有的，因此一旦确定，就可以被所有该相机拍摄的图像复用。一般来说，相机的内部参数会使用一个大小为 $3\times 3$ 的矩阵表示：

$$
\begin{align*}
M_{camera} =
\begin{bmatrix}
f_x & 0 & c_x \\\\ 0 & f_y & c_y \\\\ 0 & 0 & 1
\end{bmatrix}.
\end{align*}
$$

对于外部参数，他通常是由一些旋转和平移的向量组成，用于表示三维空间中物体位置到相机坐标位置的转换。由于每张图像中拍摄的场景都可能存在不同，外部参数往往因拍摄的图像而异。

{{< admonition type=note title="译者注：坐标转换和坐标空间" open=false >}}
这里可以参考图形学渲染管线中的顶点处理阶段进行理解。在变换过程中主要涉及五种坐标空间，目的是为了将三维空间中的物体投影到相机平面中。相关信息可以查阅 [LearnOpenGL](https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/)。
{{< /admonition >}}

对于立体视觉应用，图像畸变是首先需要被消除和矫正的，否则会引起后续处理流程的误差累计或产生错误结果。为了得到用于消除图像畸变的参数，我们需要使用相机拍摄一些精心设计场景。其中，最为常见的场景是棋盘格图像。在精心设计的场景中，我们能够预先知道真实世界中的关键点坐标信息，也可以通过图像处理的方式得到关键点在图像中的相应坐标信息。通过多个这样的信息对，就可以计算得到用于消畸变的畸变参数、相机内外参数等。一般来说，为了得到更好的结果，往往需要对相同的场景拍摄至少10张不同角度的测试图像。

## 代码详解

如第二节最后所述，我们需要至少10张测试图像用于相机标定。OpenCV的仓库中提供了一组棋盘格图像用于展示相机标定的过程，他们位于仓库的`samples/data`目录下，名称为`left*.jpg`。考虑其中任意一张棋盘格图像，我们需要知道的关键信息是一组三维世界中点的坐标和与之对应的二维图像中的坐标。其中，二维图像中点的坐标十分容易获的，他们就是棋盘格图像中任意两个白色或黑色方块的接触点。

对于三维世界中点的坐标，由于每次拍摄时相机和棋盘格图像之间的相对位置不同，我们需要知道他们的 $(x, y, z)$ 坐标。但为了简单起见，我们可以认为拍摄时棋盘格始终位于 $XY$ 坐标平面内，即所有三维世界中点的 $z$ 坐标均为0。此时，实际情况下 $z$ 坐标的变化认为是由相机沿光轴方向前后移动引起的。通过模型的简化，我们在相机标定时仅需知道三维世界中点的 $(x,y)$ 坐标即可。此外，我们还可以进一步简化这些坐标：由于棋盘格大小是已知且固定的，我们就可以使用一组二维索引表示世界中点的坐标，如 $(0, 0)$，$(0, 1)$，$(0, 2)$…… 通过索引值计算得到的结果是真实结果的缩放值。若此时可以知道棋盘格方块实际大小，就能够得到三维世界中点的真实坐标。对于本文的示例场景，由于我们并未实际拍摄图像，并不知晓棋盘格方块的大小，所以略过了棋盘格图像块大小的参数（该参数的单位一般是毫米）。

一般来说，三维世界中点的坐标被称为**物点**，二维图像中点的坐标被称为**像点**。

### 准备阶段

为了能够搜索到棋盘格图像中的像点，我们使用`cv2.findChessboardCorners()`函数进行处理。该函数需要我们指定棋盘格图像的像点排列情况，如 $8\times 8$ 或 $5 \times 5$ 大小的棋盘格。在本文的示例中，我们使用的是 $7\times 6$ 排列情况的棋盘格。通常情况下，一个$8\times 8$排列情况的棋盘格能够找到 $7 \times 7$ 个像点。函数运行后会返回一个表示是否找到指定排列模式的标志位`retval`。如果`retval`为`True`，表示函数找了指定的像点集合。并且该函数返回的像点集合会按顺序排列，例如从左到右从上到下的顺序。

{{< admonition type=note title="小提示" open=true >}}
在给定一组棋盘格图像后，该函数可能无法在任何一张图像中找到符合要求的像点。因此，较好的做法是：启动相机并拍摄图像后，就使用该函数进行像点的检测；一旦能够在拍摄的图像中找到符合要求的像点，就将结果保存到数组中。此外，在两次拍摄之间，可以预留一些时间，用于调整棋盘格的位置和方向。通过重复上述过程，直到所需的物像点对数满足要求。即使是本文使用的13张图像，我们也无法保证每张图像都能够找到物像点对。因此，我们的做法是：读取所有的图像并逐个检测，然后只选取其中能够检测出符合要求像点的图像进行后续处理。除了棋盘格标定板外，我们还可以使用圆形阵列标定板。此时，可以使用`cv2.findCirclesGrid()`函数搜索符合要求的像点，并且圆形阵列标定板可以使用更少的图像数量完成相机标定。
{{< /admonition >}}

{{< admonition type=note title="译者注：OpenCV中支持的标定板类型" open=false >}}
截止4.8.0版本，OpenCV的API中支持了棋盘格、圆形网格和CharuCo三种标定板。来源：[知乎](https://zhuanlan.zhihu.com/p/353316030)
{{< /admonition >}}

{{< admonition type=note title="译者注：棋盘格的角点排列" open=false >}}
一般来说，拍摄长宽两个方向上具有不同方块数的棋盘格。若长宽方块数量相同，得到的结果存在二义性，无法分辨是没有旋转的结果，还是旋转了$180^{\circ}$的结果。
{{< /admonition >}}

对于搜索到像点，我们可以使用`cv2.cornerSubPix()`来进一步提高他们的坐标精度。此外，还可以使用`cv2.drawChessboardCorners`在图像中绘制精细化后的有序像点集合。上述的所有流程都可以使用下面的代码完成功能实现。

```python linenums="1"
import cv2
import glob
import numpy as np

# termination criteria
criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.001)
# prepare object points, like (0,0,0), (1,0,0), (2,0,0) ....,(6,5,0)
objp = np.zeros((6 * 7, 3), np.float32)
objp[:, :2] = np.mgrid[0:7, 0:6].T.reshape(-1, 2)
# Arrays to store object points and image points from all the images.
objpoints = []  # 3d point in real world space
imgpoints = []  # 2d points in image plane.
images = glob.glob("*.jpg")
for fname in images:
    img = cv2.imread(fname)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    # Find the chess board corners
    ret, corners = cv2.findChessboardCorners(gray, (7, 6), None)
    # If found, add object points, image points (after refining them)
    if ret == True:
        objpoints.append(objp)
        corners2 = cv2.cornerSubPix(gray, corners, (11, 11), (-1, -1), criteria)
        imgpoints.append(corners2)
        # Draw and display the corners
        cv2.drawChessboardCorners(img, (7, 6), corners2, ret)
        cv2.imshow("img", img)
        cv2.waitKey(500)
cv2.destroyAllWindows()
```

对于任意一张棋盘格图像，搜索像点并将像点绘制在棋盘格图像中的结果如图2所示。

<figure>
  <img src="calib_pattern.jpg" style="width:50%;" alt="calibration pattern">
  <figcaption>
    <h4>图2 有序像点的可视化</h4>
  </figcaption>
</figure>

### 标定参数

至此，我们已经得到了用于相机的标定的关键信息：物像点对。接下来，我们将使用`cv2.calibrateCamera()`函数完成相机的标定过程，得到相机的内外参数以及畸变参数。

``` python
# ret - 标定状态
# mtx - 相机内参矩阵
# dist - 畸变参数
# rvecs - 旋转向量
# tvecs - 平移向量
ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(
    objpoints, imgpoints, gray.shape[::-1], None, None
)
```

{{< admonition type=note title="译者注：张氏标定法" open=false >}}
感兴趣的同学可以查看张正友老师于1998年发表的论文 [A Flexible New Technique for Camera Calibration](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr98-71.pdf)，其中详细解释了相机标定的过程和数学推导。
{{< /admonition >}}

### 图像消畸变

在获得了相机标定参数后，我们就能够完成图像消畸变任务了。并且，OpenCV提供了两种方式来完成这项任务：使用`cv2.undistort()`函数和使用`cv2.remap()`函数。但在此之前，我们可以使用`cv2.getOptimalNewCameraMatrix()`函数对相机内部参数和畸变参数进行优化，优化的结果由自由度参数 $\alpha$ 控制。消畸变时，会利用标定参数生成变换矩阵对图像进行修改。这一过程中，图像的边缘可能会出现黑边现象。当优化自由度参数 $\alpha=0$ 时，该函数将会返回一组能够让黑边变得最小的标定参数，此时图像的边缘可能会丢失一些含有信息的像素；当优化自由度参数 $\alpha=1$ 时，该函数不会针对黑边进行优化，此时消畸变的结果会包含黑边。此外，该函数还会返回一个图像的ROI，用于将图像的黑边裁剪移除。

现在，我们可以读取待处理图像，然后使用`cv2.getOptimalNewCameraMatrix()`函数优化相机的内部参数和畸变参数。本文使用的是名为`left12.jpg`的图像，也就是本文的图1。

```python
img = cv2.imread('left12.jpg')
h, w = img.shape[:2]
newcameramtx, roi = cv2.getOptimalNewCameraMatrix(mtx, dist, (w,h), 1, (w,h))
```

**使用`cv2.undistort()`函数**

这种方式最为简单，将优化前后的相机内参与畸变参数作为输入传递给`cv2.undistort()`函数，就可以得到消畸变后的结果。然后，再使用上文提到的ROI对消畸变结果进行裁剪。

```python linenums="1"
# undistort
dst = cv2.undistort(img, mtx, dist, None, newcameramtx)
# crop the image
x, y, w, h = roi
dst = dst[y:y+h, x:x+w]
cv2.imwrite('calibresult.png', dst)
```

**使用`cv2.remap()`函数**

这种方式与`cv2.undistort()`不同：他首先使用`cv2.initUndistortRectifyMap()`函数建立了消畸变处理前后图像的映射关系，然后使用`cv2.remap()`对待处理图像应用该映射，最终得到消畸变结果。

```python
# undistort
mapx, mapy = cv2.initUndistortRectifyMap(mtx, dist, None, newcameramtx, (w,h), 5)
dst = cv2.remap(img, mapx, mapy, cv2.INTER_LINEAR)
# crop the image
x, y, w, h = roi
dst = dst[y:y+h, x:x+w]
cv2.imwrite('calibresult.png', dst)
```

但无论使用哪种方式，都能够得到相同的消畸变结果。本文图1的消畸变结果如图3所示。可以发现：图像中原本向边缘膨胀的线条变直了。在完成相机标定和优化后，可以使用numpy中的`numpy.savez()`或`numpy.savetxt()`方法保存相机内外参数和畸变参数，以便后续读取使用。

<figure>
  <img src="calib_result.jpg" style="width:50%;" alt="calibration result">
  <figcaption>
    <h4>图3 图像消畸变结果示例</h4>
  </figcaption>
</figure>

### 重投影误差

重投影误差可以用于估计用于图像消畸变参数的好坏：重投影误差越接近于零，说明消畸变参数越精确。在计算得到相机的内外参数和畸变参数后，我们可以使用`cv2.projectPoints`函数将物点转换到二维空间得到理论像点；然后，计算理论像点和实际像点之间坐标的欧式距离；最后，计算所有理论像点和实际像点之间欧式距离的算术均值作为重投影误差。

```python linenums="1"
mean_error = 0
for i in range(len(objpoints)):
    imgpoints2, _ = cv2.projectPoints(objpoints[i], rvecs[i], tvecs[i], mtx, dist)
    error = cv2.norm(imgpoints[i], imgpoints2, cv2.NORM_L2)/len(imgpoints2)
    mean_error += error
print( "total error: {}".format(mean_error/len(objpoints)) )
```

## 完整的代码

```cpp
#include <iostream>
#include <memory>
#include <vector>

#include <opencv2/opencv.hpp>

int main() {
  std::vector<std::string> inputPaths;  // 需要给定输入图像组

  std::vector<cv::Mat> inputImages;
  inputImages.reserve(inputPaths.size());
  for (const auto& inputPath : inputPaths) {
    cv::Mat inputImage = cv::imread(inputPath);
    if (inputImage.empty()) {
      std::cout << "Input image is empty. [" << inputPath << "]\n";
      return EXIT_FAILURE;
    }
    inputImages.push_back(inputImage);
  }

  size_t imageCount = inputImages.size();

  std::vector<cv::Mat> inputGrayImages;
  inputGrayImages.reserve(inputImages.size());
  for (const auto& inputImage : inputImages) {
    cv::Mat inputGrayImage;
    cv::cvtColor(inputImage, inputGrayImage, cv::COLOR_BGR2GRAY);
    inputGrayImages.push_back(inputGrayImage);
  }

  static std::vector<cv::Point3f> objectPoint = [](const cv::Size& size) {
    std::vector<cv::Point3f> points;
    for (auto h = 0; h < size.height; h++) {
      for (auto w = 0; w < size.width; w++) {
        points.emplace_back(w, h, 0.F);
      }
    }
    return points;
  }(cv::Size(7, 6));

  std::vector<std::vector<cv::Point3f>> objectPoints;
  objectPoints.reserve(inputImages.size());
  std::vector<std::vector<cv::Point2f>> imagePoints;
  imagePoints.reserve(inputImages.size());
  const cv::Size patternSize(7, 6);
  const auto criteria = cv::TermCriteria(
      cv::TermCriteria::Type::EPS + cv::TermCriteria::Type::MAX_ITER, 30,
      0.001);
  for (size_t i = 0; i < imageCount; i++) {
    std::cout << inputPaths[i] << std::endl;
    std::vector<cv::Point2f> corners;
    auto ret =
        cv::findChessboardCorners(inputGrayImages[i], patternSize, corners);
    if (ret) {
      objectPoints.push_back(objectPoint);
      cv::cornerSubPix(inputGrayImages[i], corners, cv::Size(11, 11),
                       cv::Size(-1, -1), criteria);
      imagePoints.push_back(corners);
    }
  }

  auto imageSize = inputGrayImages.front().size();

  cv::Mat cameraMatrix;
  cv::Mat distCoeffs;
  std::vector<cv::Mat> rvecs;
  std::vector<cv::Mat> tvecs;
  auto ret = cv::calibrateCamera(objectPoints, imagePoints, imageSize,
                                 cameraMatrix, distCoeffs, rvecs, tvecs);
  std::cout << "Total error: " << ret << "\n";

  cv::Rect validPixROI;
  auto newCameraMatrix = cv::getOptimalNewCameraMatrix(
      cameraMatrix, distCoeffs, imageSize, 1.F, imageSize, &validPixROI);

  for (size_t i = 0; i < imageCount; i++) {
    cv::Mat undistortImage;
    cv::undistort(inputImages[i], undistortImage, cameraMatrix, distCoeffs,
                  newCameraMatrix);
    std::string outputImageName =
        "undistortImage_" + std::to_string(i) + ".png";
    cv::imwrite(outputImageName, undistortImage(validPixROI));
  }

  float meanError = 0.F;
  for (size_t i = 0; i < objectPoints.size(); i++) {
    std::vector<cv::Point2f> objectProjectToImagePoints;
    cv::projectPoints(objectPoints[i], rvecs[i], tvecs[i], cameraMatrix,
                      distCoeffs, objectProjectToImagePoints);
    auto error =
        cv::norm(imagePoints[i], objectProjectToImagePoints, cv::NORM_L2);
    error /= objectProjectToImagePoints.size();
    meanError += error;
  }

  meanError /= objectPoints.size();
  std::cout << "Total error: " << ret << "\n";

  return 0;
}
```

```python
import cv2
import glob
import numpy as np
from pprint import pprint

criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.001)

obj_point = np.zeros((6 * 7, 3), np.float32)
obj_point[:, :2] = np.mgrid[0:7, 0:6].T.reshape(-1, 2)

obj_points = []
img_points = []

image_names = []  # 需要给定输入图像组

(w, h) = (0, 0)

for image_name in image_names:
    pprint("Current image: [{}]".format(image_name))
    image = cv2.imread(image_name)
    image_gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    (w, h) = image_gray.shape[::-1]

    ret, corners = cv2.findChessboardCorners(image_gray, (7, 6), None)
    # ret, corners = cv2.findCirclesGrid(image_gray, (7, 6), None)
    if ret is True:
        obj_points.append(obj_point)
        corners_refined = cv2.cornerSubPix(
            image_gray, corners, (11, 11), (-1, -1), criteria
        )
        img_points.append(corners_refined)

pprint("Image w = {}, h = {}".format(w, h))
ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(
    obj_points, img_points, (w, h), None, None
)

# undistortion
image = cv2.imread(image_names[0])
new_camera_mtx, roi = cv2.getOptimalNewCameraMatrix(mtx, dist, (w, h), 1, (w, h))
dst = cv2.undistort(image, mtx, dist, None, new_camera_mtx)

# x, y, w, h = roi
# dst = dst[y: y+h, x: x+w]
cv2.imwrite("calibresult.png", dst)

# reprojection error
mean_error = 0
for i in range(len(obj_points)):
    img_points2, _ = cv2.projectPoints(obj_points[i], rvecs[i], tvecs[i], mtx, dist)
    error = cv2.norm(img_points[i], img_points2, cv2.NORM_L2) / len(img_points2)
    print(error)
    mean_error += error
pprint("Total error: {}".format(mean_error / len(obj_points)))
```

