# 3D Orientation Tracking and Panorama Reconstruction With IMU

Estimating a rotating camera's 3D orientation from IMU data alone, then using those orientations to stitch the camera frames into a 360° panorama without any image feature matching.

📄 Full write-up: [report.pdf](report.pdf) (UCSD ECE276A, Project 1)

<table>
  <tr>
    <td><img src="results/orientation/orientation_1.png" alt="Dataset 1 orientation: VICON vs PGD vs gyro integration"></td>
    <td><img src="results/panorama/panorama_nov_1.png" alt="Dataset 1 panorama from IMU-estimated orientation"></td>
  </tr>
  <tr>
    <td align="center"><sub>Dataset 1: roll / pitch / yaw from the optimizer (red) against VICON ground truth (blue) and raw gyro integration (green)</sub></td>
    <td align="center"><sub>Dataset 1: panorama built only from the IMU-estimated orientation</sub></td>
  </tr>
</table>

## Overview

Feature-based panorama stitching is unreliable in many robotics settings: scenes can lack texture, lighting changes, and motion blur breaks correspondences. This project avoids images entirely during alignment. Instead it:

1. Estimates the camera's orientation over time as a trajectory of **unit quaternions**, using the gyroscope and accelerometer.
2. Uses each frame's orientation to project its pixels onto a sphere in the world frame and unrolls that sphere into an **equirectangular panorama**.

## Method

**IMU calibration.** The first few seconds of each recording are assumed static, so the accelerometer should read pure gravity. The accelerometer bias is the maximum-likelihood estimate under Gaussian noise (the mean offset from $[0, 0, g]$ over that window), and it is subtracted from every reading.

**Gyroscope integration (initial guess).** Starting from the identity quaternion, angular velocity is integrated with the quaternion kinematics model

$$q_{t+1} = q_t \circ \exp\big([0,\ \tfrac{\Delta\tau_t}{2}\,\omega_t]\big)$$

This drifts over time, so it only serves as the starting point.

**Projected gradient descent (refinement).** The trajectory $q_{1:T}$ is refined by minimizing a cost with two terms:

- **Motion term:** how far each $q_{t+1}$ deviates from what the gyroscope predicts from $q_t$.
- **Observation term:** how far the measured acceleration deviates from gravity $[0,0,0,1]$ rotated into the body frame by $q_t$.

$$c(q_{1:T}) = \tfrac{1}{2}\sum_t \big\lVert 2\log\big(q_{t+1}^{-1} \circ f(q_t, \Delta\tau_t\omega_t)\big)\big\rVert^2 + \tfrac{1}{2}\sum_t \big\lVert [0, a_t] - q_t^{-1} \circ [0,0,0,1] \circ q_t \big\rVert^2$$

Gradients come from autograd. After each step every quaternion is renormalized to stay a valid rotation, and iteration stops once the change in cost falls below a threshold.

**Panorama construction.** Each camera frame is matched to the most recent IMU timestamp. Every pixel is turned into a ray from the camera's horizontal and vertical field of view, rotated into the world frame, converted to longitude and latitude, and written into the matching pixel of the equirectangular panorama.

## Results

Datasets 1–9 include VICON motion-capture ground truth. Datasets 10–11 are held-out recordings with IMU and camera data only.

### Orientation tracking

Each plot compares roll, pitch and yaw for **VICON** (blue), the **PGD estimate** (red) and **raw gyro integration** (green). The final PGD cost is in each plot title.

| | | |
|:-:|:-:|:-:|
| <img src="results/orientation/orientation_2.png"> | <img src="results/orientation/orientation_3.png"> | <img src="results/orientation/orientation_4.png"> |
| Dataset 2 (cost 5.66) | Dataset 3 (cost 6.99) | Dataset 4 (cost 18.53) |
| <img src="results/orientation/orientation_5.png"> | <img src="results/orientation/orientation_6.png"> | <img src="results/orientation/orientation_7.png"> |
| Dataset 5 (cost 15.13) | Dataset 6 (cost 11.69) | Dataset 7 (cost 22.31) |
| <img src="results/orientation/orientation_8.png"> | <img src="results/orientation/orientation_9.png"> | <img src="results/orientation/orientation_10.png"> |
| Dataset 8 (cost 9.91) | Dataset 9 (cost 9.38) | Dataset 10 (no ground truth) |

Dataset 9 shows the effect most clearly: gyro integration drifts by tens of degrees in roll and pitch, while the PGD estimate stays on top of VICON.

### Panoramas: IMU estimate vs. VICON

The left column uses the PGD orientation estimate and the right column uses VICON ground truth with the same frames.

| PGD estimate (IMU only) | VICON ground truth |
|:-:|:-:|
| <img src="results/panorama/panorama_nov_1.png"> | <img src="results/panorama/panorama_vic_1.png"> |
| <img src="results/panorama/panorama_nov_2.png"> | <img src="results/panorama/panorama_vic_2.png"> |
| <img src="results/panorama/panorama_nov_8.png"> | <img src="results/panorama/panorama_vic_8.png"> |
| <img src="results/panorama/panorama_nov_9.png"> | <img src="results/panorama/panorama_vic_9.png"> |

### Held-out datasets (no ground truth)

| Dataset 10 | Dataset 11 |
|:-:|:-:|
| <img src="results/panorama/panorama_nov_10.png"> | <img src="results/panorama/panorama_nov_11.png"> |
| <img src="results/orientation/orientation_10.png"> | <img src="results/orientation/orientation_11.png"> |

### Optimizer convergence

On every dataset the cost falls steeply within the first ~50 iterations and converges in about 200.

<details>
<summary>Cost history for all 11 datasets</summary>

| | | |
|:-:|:-:|:-:|
| <img src="results/cost/cost_1.png"> | <img src="results/cost/cost_2.png"> | <img src="results/cost/cost_3.png"> |
| <img src="results/cost/cost_4.png"> | <img src="results/cost/cost_5.png"> | <img src="results/cost/cost_6.png"> |
| <img src="results/cost/cost_7.png"> | <img src="results/cost/cost_8.png"> | <img src="results/cost/cost_9.png"> |
| <img src="results/cost/cost_10.png"> | <img src="results/cost/cost_11.png"> | |

</details>

## Takeaways

- **Roll and pitch are corrected well.** Projected gradient descent removes most of the gyroscope drift, and the IMU-only panoramas closely match the ones built from VICON.
- **Yaw is not observable from gravity.** Rotating about the vertical axis doesn't change the direction of gravity, so the accelerometer term carries no yaw information. Yaw falls back on the gyroscope and still drifts (e.g. dataset 4). A magnetometer or visual cue would be needed to fix it.
- **The estimates are noisy.** Optimizing every quaternion independently in one batch lets small conflicts between the motion and observation terms show up as jitter, which appears as ghosting in the panoramas. Adding a smoothness prior or using a filter-based estimator would help.
- **Stitching is naive by design.** Pixels are written straight into the panorama with no blending or seam handling, so visible cuts appear where frames overlap.

## Repository Layout

```
report.pdf             full technical report
results/orientation/   roll/pitch/yaw comparison plots, datasets 1–11
results/panorama/      panoramas: panorama_nov_* (IMU estimate), panorama_vic_* (VICON)
results/cost/          PGD cost-vs-iteration curves, datasets 1–11
```

## License

[MIT](LICENSE)
