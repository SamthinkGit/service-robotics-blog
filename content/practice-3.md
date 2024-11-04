**Title: Robust Autonomous Parallel Parking System**

In this project, autonomous car must perform parallel parking on the right-hand side of a typical street. The car's dynamic is similar to that of a real vehicle, with rear-wheel drive and Ackermann steering through two front wheels. It uses V and W commands to control speed and steering angle, respectively, while relying on three laser sensors for distance measurements: one in front, one on the right, and one on the rear. Importantly, GPS data is unavailable, except for the yaw value that functions similarly to a compass. The main objective was to create robust software that allows the car to successfully park, even in varied scenarios.

Here, It is explained an approach to accomplish this task:

### 1. Navigation Strategy

Since we cannot rely on detecting the nearby cars or walls directly, the global navigation strategy must set a clear direction along which the vehicle moves. To achieve robustness, I made sure that all movements were **relative to the direction of the street** and avoided using global coordinates—purely relying on relative turns. This street direction was defined by a constant, `INITIAL_YAW`, which allowed the vehicle to drive along a set path. 

<div align="center">
    <img src="./media/p3/yaw-direction.png" height=200px>
</div>

The vehicle moved at a **reasonable speed** while using the right laser sensor to **scan for an available parking spot**, ensuring it was large enough for the car plus a small extra margin.

### 2. Parking Procedure

For parking, I implemented a typical parallel parking maneuver. The steps were as follows:

- **Advance Past the Spot**: Once the desired parking space was identified, the car moved slightly forward until the space was aligned parallel behind the vehicle.

- **Clockwise Turn**: The car began to reverse, turning clockwise into the spot until reaching a 60-degree angle relative to the direction of the street. If there was a car parked on the right side, the car would rotate only as far as possible while **keeping a safe distance from the obstacle**.

- **Counter-Clockwise Turn**: Next, the car turned counter-clockwise to align itself parallel with the street. If there was a car on the right, it advanced cautiously to avoid a collision. Similarly, **if another car was parked behind**, it would halt once it could not safely reverse further.

- **Adjustment**: Finally, the car moved forward until it maintained a reasonable distance from the car in front (if there was one) or until it had advanced one meter from the previous position.

<div align="center">
    <img src="./media/p3/parking.png" height=200px>
</div>

### Final Results

This approach proved successful, allowing the car to park effectively in both the standard scenario and in more complex variations where other vehicles needed to be avoided. The implementation of relative-only navigation helped the system adapt to different street orientations without the need for global coordinates, while the detailed parking maneuver ensured precise placement regardless of the surroundings.
