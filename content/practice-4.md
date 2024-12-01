# Practice: Warehouse Robot Navigation System

This exercise focuses on implementing a **navigation system for a warehouse robot**, where the main goal is to move shelves from one location to another within a warehouse, similar to how Amazon's logistics robots operate. The robot needs to navigate through the environment using **route planning** provided by the **Open Motion Planning Library (OMPL)** and must execute that plan effectively, adjusting for both circular and rectangular geometries depending on the context.

This solution is divided into multiple key components:

1. **Registration of World Coordinates to Pixel Coordinates**
2. **Path Planning with OMPL**
3. **State Validation**
4. **Warehouse Map Management and Visualization**
5. **Special Considerations for Safe Navigation**

Let’s explore the implementation of each part.

## 1. Registration of World Coordinates to Pixel Coordinates 🌍➡️🖼️

The **registration of world coordinates to pixel coordinates** is handled by the `Coordinate.from_world_coordinates()` method. This method uses a transformation matrix to convert the robot's real-world position into corresponding pixel positions on the map. This matrix has been hand-crafted by moving the robot throughout the map and comparing the obtained coordinates. 

## 2. Path Planning with OMPL ➕

The core of the robot's navigation is achieved through the **(OMPL)**. The planning process uses **RRTConnect** to find an optimal path from the robot's current position to its desired target, whether it is picking up or dropping off a shelf.

To do this, we set up a **3D State Space** (x, y, yaw) representing the warehouse grid, and then bounds it to the map's dimensions.  The **`plan()`** method of the `OmplManager` class is responsible for generating this path, from the robot's current **source coordinate** to the desired **target coordinate**. If a valid path is found, it returns a list of coordinates representing the waypoints that the robot should follow to reach its destination.

> This is implemented in the `OmplManager` class, specifically within the `plan()` method.

## 3. State Validation ✅


To ensure that the planned paths are valid, OMPL uses a **state validity checker** (`_is_state_valid()`), which verifies that the proposed robot states do not overlap with any obstacles. This is crucial to prevent collisions with shelves, walls, or other barriers within the warehouse.

The **state validity checker** works by simulating the robot's position on the map and checking for collisions. It draws a square or a rectangle on the map in order to emulate the size of the robot. If any obstacle is found within this bounds.

For determining the rotation of the state we compute a subpath between the latest state and the current state. The rotation is equivalent to the direction of the path. This enables the planner to consider **yaw rotation** of the robot when defining if the state is valid.

<div align="center">
    <img src="./media/p4/path_planning.png" height=200px>
</div>

## 4. Warehouse Map Management and Visualization 🖼

The **warehouse environment** is represented as a 2D map, where the robot must navigate while avoiding obstacles. The `Map` class is responsible for managing and visualizing this environment, with tools for showing the robot's path, indicating key locations, and marking obstacles.

- **Map Initialization**: The map is created using an image file that represents the warehouse layout (`Map()`), with different colors used to indicate **occupied**, **free**, and **path** areas.

- **Key Visual Elements**: The `Map` class includes methods for drawing visual indicators, such as **arrows** (to show robot movement direction), **circles** (to mark waypoints), and **keypoints** (indicating specific important spots like targets).

- **Dynamic Updates**: As the robot navigates, the map is dynamically updated to reflect the robot's current position and orientation, and any shelves being moved. This provides a visual reference to ensure that the robot is accurately following the planned path.

The **GUI** allows for real-time visualization, giving insight into how the robot makes decisions and how its planned path adapts to obstacles in the environment.

> This is implemented through the `Map` class, with core drawing methods like `arrow()`, `circle()`, and `connect()` to display the robot's actions and path on the map.

## Challenges and Observations ✨

- **Geometry Handling**: Adjusting the robot's geometry when lifting and carrying shelves required careful consideration to ensure paths were safe and valid, ensuring that the state validity checks adapted accordingly.

- **Visualization**: Providing real-time visualization of the robot's state, path, and surroundings greatly improved the debugging process, making it easier to identify issues like incorrect rotations or collisions.

# Results

<div align="center">
    <video width="600" controls>
        <source src="https://github.com/user-attachments/assets/2ecc8b7f-bf99-46c3-9f17-ed443cfb675d" type="video/mp4">
    </video>
</div>
