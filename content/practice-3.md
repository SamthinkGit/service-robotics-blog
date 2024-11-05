**Title: Robust Autonomous Parallel Parking System**

In this project, autonomous car must perform parallel parking on the right-hand side of a typical street. The car's dynamic is similar to that of a real vehicle, with rear-wheel drive and Ackermann steering through two front wheels. It uses V and W commands to control speed and steering angle, respectively, while relying on three laser sensors for distance measurements: one in front, one on the right, and one on the rear. Importantly, GPS data is unavailable, except for the yaw value that functions similarly to a compass. The main objective was to create robust software that allows the car to successfully park, even in varied scenarios.

Here, It is explained an approach to accomplish this task:

### 1. Navigation Strategy

To navigate in the correct direction, we first need to detect the street’s orientation. To achieve this, we'll move the car forward while comparing two points on the car's lateral laser sensor. By calculating the **slope** of the values detected at these two points, we can determine a value that indicates the street’s inclination relative to the car. As the car moves, we monitor this slope and record the car’s yaw as `initial_yaw` once the slope reaches zero, which **signifies alignment with the street direction**.

<div align="center">
    <img src="./media/p3/pending.png" height=200px>
</div>

To achieve robustness, we made sure that all movements were **relative to the direction of the street** and avoided using global coordinates—purely relying on relative turns.

<div align="center">
    <img src="./media/p3/yaw-direction.png" height=200px>
</div>

Additionally, the vehicle moved at a **reasonable speed** while using the right laser sensor to **scan for an available parking spot**, ensuring it was large enough for the car plus a small extra margin.

### 2. Detecting an Spot
For detecting an available spot for parking we translated all laser into vector values. Then, we determine if all vectors are outside of a box-sized umbral. If so, then there is an available space.

<div align="center">
    <img src="./media/p3/vectors.png" height=200px>
</div>

### 3. Parking Procedure

For parking, it has been implemented a typical parallel parking maneuver. The steps were as follows:

- **Advance Past the Spot**: Once the desired parking space was identified, the car moved slightly forward until the space was aligned parallel behind the vehicle.

- **Clockwise Turn**: The car began to reverse, turning clockwise into the spot until reaching a 60-degree angle relative to the direction of the street. If there was a car parked on the right side, the car would rotate only as far as possible while **keeping a safe distance from the obstacle**.

- **Counter-Clockwise Turn**: Next, the car turned counter-clockwise to align itself parallel with the street. If there was a car on the right, it advanced cautiously to avoid a collision. Similarly, **if another car was parked behind**, it would halt once it could not safely reverse further.

- **Adjustment**: Finally, the car moved forward until it maintained a reasonable distance from the car in front (if there was one) or until it had advanced one meter from the previous position.

<div align="center">
    <img src="./media/p3/parking.png" height=200px>
</div>

### Final Results

This approach proved successful, allowing the car to park effectively in both the standard scenario and in more complex variations where other vehicles needed to be avoided. The implementation of relative-only navigation helped the system adapt to different street orientations without the need for global coordinates, while the detailed parking maneuver ensured precise placement regardless of the surroundings.
