# 🧩 Visual Localization of a Robot with AprilTags

This document describes the implementation of a visual localization system for a domestic service robot, using AprilTags visual markers and the OpenCV library. The system enables the robot to estimate its absolute position in a controlled environment by detecting and interpreting the visual beacons present in the scene.

---

## Tag Wrapper Implementation

To simplify the processing of detected tags, a wrapper class `Tag` has been implemented. This class encapsulates all relevant information and operations for each detected tag, including:

| **Feature**               | **Description**                                                                 |
|---------------------------|-------------------------------------------------------------------------------|
| **Attributes**            | Stores information such as position, yaw angle, and corners of the tag.        |
| **Transformation Methods**| Provides methods for computing transformations between different reference systems. |

## 🎤 Localization Algorithm

The localization process includes the following main steps:

### 1. **Beacon Detection**
The `pyapriltags` library is used to detect visual markers in the image captured by the camera.

```python
@classmethod
Tag.detect(cls, image) -> list["Tag"]:
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    tags = detector.detect(gray)
    return [Tag(content=tag, image=image) for tag in tags]
```

### 2. **Relative Estimation (PnP)**
The **Perspective-n-Point (PnP)** technique is applied to calculate the position and orientation of the camera relative to each detected beacon.

```python
success, rvec, tvec = cv2.solvePnP(
    tag_points,
    self.corners,
    matrix_camera,
    dist_coeffs,
    flags=cv2.SOLVEPNP_IPPE_SQUARE,
)
```

| **Parameter**       | **Description**                                     |
|---------------------|---------------------------------------------------|
| `tag_points`        | 3D coordinates of the marker's vertices.          |
| `self.corners`      | 2D coordinates of the detected vertices in the image.|
| `matrix_camera`     | Camera's intrinsic matrix.                        |
| `dist_coeffs`       | Camera's distortion coefficients.                  |
| `rvec`, `tvec`      | Estimated rotation and translation vectors.        |

### 3. **Reference System Transformations**
Transformations between the reference systems of the camera, the beacons, and the absolute world are calculated.

## 🌍 Transformations for Robot Localization

To estimate the robot's absolute position, the following transformations are performed:

1. **World to Tag Transformation**: Converts the absolute world coordinates to the local reference frame of the detected tag. This is achieved using the tag's position and orientation (yaw), which define the translation and rotation required to move from the world reference frame to the tag's reference frame.

    **Matrix Design**:
    ```python
    def world2tag(self):
        alpha = self.yaw  # <- This is the yaw of the TAG
        T = np.array([
            [np.cos(alpha), -np.sin(alpha), 0, self.position[0]],
            [np.sin(alpha),  np.cos(alpha), 0, self.position[1]],
            [0,              0,             1, TAG_HEIGHT],
            [0,              0,             0, 1]
        ])
        return T
    ```

2. **Axis Alignment**: Adjusts the axes from the global's reference frame to match the OpenCV's expected coordinate system. 

    **Matrix Design**:
    ```python
    R = np.array([
        [0, 0, 1, 0],
        [-1, 0, 0, 0],
        [0, -1, 0, 0],
        [0, 0,  0, 1]
    ])
    ```

3. **Tag to Robot Transformation**: Converts the adjusted tag coordinates into the robot's local reference frame by inverting the transformation from the camera to the tag. This involves applying the tag-to-camera matrix and then the axis alignment.

    **Matrix Design**:
    ```python
    def tag2camera(self):
        # PnP detection...

        R, _ = cv2.Rodrigues(rvec)
        T = np.eye(4)
        T[:3, :3] = R
        T[:3, 3] = tvec.ravel()
        return np.linalg.inv(T)  # We invert the obtained matrix
    ```

### Combined Transformation
The combined transformation sequence calculates the robot's position and orientation relative to the world:

```python
world2tag = tag.world2tag()
tag2camera = tag.tag2camera()
M = world2tag @ (R @ tag2camera)  # Combines all transformations
```

This ensures precise mapping of the robot's position and orientation within the absolute coordinate system.

## 🔎 Transformation Simplification

The transformation matrix from the camera to the robot has been considered as negligible in this implementation. The effect of this transformation on the overall accuracy is **minimal** and its inclusion adds unnecessary complexity to the calculations. 

```python
robot2camera = [
    [1, 0, 0, 0.069],   # E.g. It only affects on ~6cm (negligible)
    [0, 1, 0, -0.047], 
    [0, 0, 1, -0.107],
    [0, 0, 0, 1]
]
```

---


## Results

<div align="center">
    <video width="600" controls>
        <source src="https://github.com/user-attachments/assets/21470418-0e02-4b4b-8134-a17881977b01" type="video/mp4">
    </video>
</div>
