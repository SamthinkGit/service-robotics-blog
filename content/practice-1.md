# 🏠 Practice 1: High-End Vacuum Cleaner

The main goal of this practice is to clean a house using a high-end vacuum cleaner. This involves completing 4 main tasks:

1. Generate a translation between robotic coordinates and map coordinates.
2. Build a grid to log which parts of the house have been cleaned.
3. Create a navigation algorithm to move through grid cells (or map points).
4. Use **BSA** as a coverage algorithm to clean the entire house.

Let's explore each task in detail in a more comprehensive manner.

---

## 📍 1. Translating Robotic Coordinates into Map Coordinates

Using the function `GUI.getMap()`, we obtain a **1025x1024** image of the house, which represents the map coordinates as pixel positions. Since our goal is to travel and perform computations only in map coordinates, we need to convert every robotic coordinate obtained from `HAL.getPose3d()` into map coordinates. This ensures that all operations, such as navigation and cleaning, can be conducted using a consistent reference frame that corresponds to the actual layout of the house.

This conversion is achieved through a transformation matrix similar to the following:

<div align="center">
    <img src="https://github.com/user-attachments/assets/49145737-6acc-4fe3-848d-6ce316d3017e" height="120px" alt="point-relation">
</div>

To solve this matrix, we need to find values for:

- **tx, ty, tz**: Translations along the x, y, and z axes.
- **alpha**: Rotation.
- **Scale**: Scaling factor.

### 1.1 Finding tx and ty

First, we determine the conversion of three points by moving the robot and recording both the robot's coordinates and the corresponding map coordinates. By doing this, we can accurately determine how the robot's coordinate system aligns with the map's coordinate system, which is essential for precise navigation.

| Position                | Robot Coordinates (Pose3D)      | Map Coordinates |
| ----------------------- | ------------------------------- | --------------- |
| **Left-Bottom Corner**  | x: 5.1316, y: 5.5555, z: 0.0009 | [996, 48]       |
| **Right-Bottom Corner** | x: -2.527, y: 5.5310, z: 0.0009 | [996, 850]      |
| **Center of Map**       | x: 0, y: 0, z: 0.0009           | [670, 570]      |

We use **OpenCV** to automatically find the **affine transformation matrix** that converts robot coordinates to map coordinates. This transformation matrix uses three pairs of corresponding points to estimate the relationship between the robot's position and the map, allowing us to correctly translate between these two coordinate systems. This approach helps ensure that the robot can accurately determine its location within the house at any given time:

```python
points_A = np.array([[5.1316, 5.5555], [-2.527, 5.5310], [0, 0]], dtype=np.float32)
points_B = np.array([[996, 48], [996, 850], [670, 570]], dtype=np.float32)
M = cv2.getAffineTransform(points_A, points_B)
```

Using the tx and ty extracted from this approximation, and adjusting these values to minimize the error, we determine the position freedom degree of the robot. This process is crucial to ensure the robot can navigate effectively without deviating from its intended path.

> Since tz is constant for all map positions, it can be ignored. From now on, we use a **3x4** matrix because we only need to represent transformations in the x and y axes, while the z-axis remains fixed. 

### 1.2 Finding alpha

We determine the angle between the horizontal axis by moving the robot along a horizontal line. Given that alpha is **180º**, we can easily deduce this value by observing how coordinate points reverse when plotted. This step allows us to align the robot's orientation with the map's reference frame, which is necessary for accurate movement and cleaning operations.

<div align="center">
    <img src="https://github.com/user-attachments/assets/d839a4e8-30a4-4ccd-bef1-872bfbc8082c" height="400px" alt="point-relation">
</div>


### 1.3 Finding the Scale

Since horizontal movement of the robot results in aligned but reversed pixel positions, we calculate the scale by comparing distances. The scale factor is critical for converting real-world distances into pixel distances, ensuring that the robot's movements are proportional to the map representation:

```python
  robot: 5.1316m - (-2.527m) = 7.6586m
  pixels: 850 - 48 = 802pixels
  scale = 802/7.6586 = 104.71
```

Finally, we use the obtained values to translate coordinates, allowing the robot to seamlessly navigate within the mapped environment without discrepancies between its internal coordinates and the map.

> A class `Map` has been created with functions like `robot2map()` for translation and `add_keypoint()` for debugging purposes. This class makes it easier to handle coordinate conversions and visualize key points on the map for troubleshooting and validation.

---

## 🗺️ 2. Building the Grid

### How It Has Been Designed

The `Grid` class is used to generate a grid over the map image. The key strength lies in the **`Cell`** dataclass, which holds information about itself and adjacent cells. This allows us to gather environment information by simply looking at the current cell. Each cell in the grid provides context not only about its own state but also about the surrounding cells, which is crucial for effective navigation and decision-making.

The grid's main responsibility is to analyze the obtained map and provide functions to determine the current cell of the robot or to extract cells sequentially. This allows the robot to systematically cover the entire area, ensuring that no spots are missed during the cleaning process.

```python
@dataclass
class Cell:

    # Grid coordinates
    i: int
    j: int

    # Pixel coordinates
    x0: int
    y0: int
    center_x: Optional[int] = None
    center_y: Optional[int] = None
    content: Optional[np.ndarray] = field(default=None, repr=False)

    # Adjacent cells
    bottom: Optional["Cell"] = field(default=None, repr=False)
    top: Optional["Cell"] = field(default=None, repr=False)
    left: Optional["Cell"] = field(default=None, repr=False)
    right: Optional["Cell"] = field(default=None, repr=False)
    _from: Optional["Cell"] = field(default=None, repr=False)
```

### Notes and Improvements

- Each cell contains a **15x15** region of the map in its `content` property, determined through trial and error to provide an optimal balance between resolution and coverage.

- A cell is defined as occupied if it contains at least one black pixel, which represents obstacles or walls that the robot must avoid.

- **Cell status is determined by color**: black or red indicates occupied, white means clean, etc. This color-coding helps the robot quickly determine the status of each cell and decide its next action.

- The `Grid.dilate_walls()` function was added to create a "danger zone" near walls, making those cells occupied and preventing the robot from crashing into walls. This additional buffer zone helps account for potential inaccuracies in positioning and ensures safer navigation.

<div align="center">
    <img src="https://github.com/user-attachments/assets/e97b19dd-986d-44cc-a4c8-fd53ac12948d" height="200px" alt="Map without/with dilatation">
</div>

---

## 🚗 3. Building Navigation Algorithms

Once we have the coordinates and a map for informed localization, we built two different navigation algorithms to ensure efficient and effective movement throughout the house:

### 3.1 Short Distance Navigation

This function is designed for short distances where we can ignore obstacles, leaving higher layers to handle them. It is useful for minor adjustments and movements within a relatively obstacle-free environment. The steps are:

1. **Rotate** until the robot faces the desired direction, then compute the shortest angle (left or right) to the target direction. This ensures minimal time and energy consumption during rotation.

2. **Move Forward** while controlling direction to correct any noise in the actuators. This step allows the robot to reach its target accurately without deviating due to robot noise.

```python

# First, point to the target
while abs(rotation) > ROTATION_ACCURACY:

    # Rotate until the yaw is appropriate
    rotation = shortest_angle_distance_radians(current_yaw, target_yaw)
    current_yaw = HAL.getPose3d().yaw
    HAL.setW(rotation)
    yield

# Then, continue with the rotation controled until reaching the point
while error > ERROR_DISTANCE:

    # Get the control attributes
    current_x, current_y = get_current_map_coords()
    current_yaw = HAL.getPose3d().yaw
    target_yaw = get_current_yaw()
    rotation = shortest_angle_distance_radians()

    # Move and recover angle
    HAL.setW(rotation)
    HAL.setV(velocity)
    yield
```

> Note: Higher abstraction layers define whether the task fails or not, allowing for more complex decision-making at higher levels of the control system.

### 3.2 Long Distance Navigation

When the robot is blocked or needs to navigate through a more complex environment, it must find new rooms to clean. We use a modified **A\*** approach to compute a path that avoids obstacles. This algorithm is well-suited for navigating around obstacles, ensuring that the robot can reach its destination even in cluttered environments.

An example of a generated path:

```python
# Build a map
mock_image = np.zeros([100, 100], dtype=int) # The map
mock_image[15:40, 15:50] = 1                 # The obstacle

# Generate a grid
grid = Grid()
grid.load_matrix(mock_image)

# Get the path from A to B
path = grid.compute_path(grid[20, 20], grid[1,1])
```

<div align="center">
    <img src="https://github.com/user-attachments/assets/e6e17490-c0c4-4256-9396-377ac2dfdc3b" height="400px" alt="A star path example">
</div>

> The `simplify_route()` function allows using short-distance navigation between path vertices, preventing stops at each traveled cell. This optimization reduces the overall travel time and makes the navigation smoother.

---

## 🌀 4. Building the BSA Algorithm

With stable navigation and localization algorithms, building a **BSA** (Backtracking Spiral Algorithm) coverage algorithm becomes straightforward. This algorithm is designed to ensure that the entire area is covered efficiently, with minimal redundancy.

> The code contains a version of the BSA algorithm that doesn't pre-plan but instead plans while moving. However, we will focus on the `bsa_v2` algorithm, which offers a good balance between speed and coverage, by first planning and thene executing.

### 4.1 Planification

BSA works by selecting adjacent cells using a priority system, marking them as cleaned, and re-planning when blocked. This method ensures that the robot systematically covers all areas, only deviating when it encounters an obstacle that requires re-planning.

```python
while True:

    # Check right
    if not current_cell.right.occupied and not current_cell.right.clean:
        result.append(current_cell.right)

    # Check top
    elif not current_cell.top.occupied and not current_cell.top.clean:
        result.append(current_cell.top)

    # Check bottom
    elif not current_cell.bottom.occupied and not current_cell.bottom.clean:
        result.append(current_cell.bottom)

    # Check left
    elif not current_cell.left.occupied and not current_cell.left.clean:
        result.append(current_cell.left)
```

<div align="center">
    <img src="https://github.com/user-attachments/assets/e11aefd2-b969-47f9-bf10-b2d28cedfe71" height="300px" alt="BSA route path">
</div>

### 🧩 Putting It All Together

With all components ready, the main loop runs BSA until blocked, then re-plans and executes the path, ensuring the robot stays on the path. If needed, the robot starts a recovery by restarting the planification. This ensures that the robot can handle unexpected situations, such as obstacles that were not initially detected, and continue cleaning effectively.

```python
# Pseudocode of bsa

while True:

    if path[0] == "plan":
        plan = bsa_planification(grid, mapping)
        path = simplify_route(plan)
        path.append("block")

    if path[0] == "block":
        target_cell = find_closest_unclean_cell(grid, mapping)
        if target_cell is None:
            break

        path_to_target = compute_path(current_cell, target_cell)
        path = simplify_route(path_to_target)
        path.append("plan")

    if needs_to_recover():
        path = ["block"]
        continue

    navigate_to(path[0])
    path.pop(0)
```

---

🎥 **[Video]**
