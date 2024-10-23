# Practice 2: Rescue People 🚁

The main goal of this practice is to build a coverage algorithm for a drone that will search for and obtain the coordinates of people lost at sea. When the drone runs out of battery or completes its search area, it will return to home.

For building this practice, we need to design 3 core components:

1. Define the coordinate **translation** between UTM and relative coordinates (drone)
2. Design a **coverage algorithm** to search for the targets
3. Design a **vision algorithm** to detect people with the camera of the drone

## 📍 1. Translating UTM Coordinates into Relative Coordinates

In this step, we first take the UTM coordinates of the origin of the drone and the target destination obtained from the exercise description.

```python
origin="40º16’48.2” N, 3º49’03.5” W."
destination="40º16’47.23” N, 3º49’01.78” W."
```

With these coordinates, we can use an [online converter](http://rcn.montana.edu/Resources/Converter.aspx) to translate them into meters. Then, we can calculate the distance in meters and use it as relative coordinates for our drone.

> From now on, we will use the drone's internal relative coordinates to navigate, only converting back the relative coordinates to UTM for printing the position of the targets.

## 🗺️ 2. Defining the Navigation

The coverage algorithm selected in this practice is the **squared spiral**, centered on the destination. This algorithm was selected due to two main reasons:

<div align="center">
    <img src="./media/p2/spiral.png" height="150px" alt="squared-spiral vs other algorithms">
</div>

1. **Easy to implement**: We do not need to build a map or other complex structures to cover the area.
2. **Center Focused**: By navigating only in a square or other shapes, we would first need to move away from the destination. With the spiral, we can start the coverage from the center (the most relevant point).

To implement this algorithm, we used a **grid-based approach**, where we estimate the size of the area covered by the drone's camera at a specific height and use it as cells. With this grid, an automatic plan for the route is created before starting the program, so the focus remains on following the route when searching for the targets.

## 👀 3. Vision

To detect the targets on the ocean, we used the **Haar Cascades** method from OpenCV. In addition to this technique, we applied several **rotations** to the image to detect targets in different orientations. With this methodology, we can easily detect the targets, using the current position of the drone as an estimation of the target's position.

<div align="center">
    <img src="./media/p2/detection.png" height="220px" alt="Detections">
</div>

For debugging and emulation purposes, we defined a **GUI** with relevant information that will be displayed to the developer. Additionally, we emulated a **battery** for the drone using a timer that starts decaying from the beginning of the program's execution.

<div align="center">
    <img src="./media/p2/GUI.png" height="220px" alt="GUI">
</div>

# 🏁 Results

<div align="center">
    <video width="600" controls>
        <source src="https://github.com/user-attachments/assets/41eea646-7be9-48d1-8203-5cf4009731c5" type="video/mp4">
    </video>
</div>

