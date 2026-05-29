# IMU and Camera Calibration

This repository documents the calibration workflow for an IMU-camera system.

The complete calibration process includes two main parts:

1. IMU noise parameter calibration using Allan variance analysis.
2. IMU-camera extrinsic calibration using Kalibr.

The final goal is to obtain both the IMU noise parameters and the rigid-body transformation between the IMU frame and the camera frame, which are required for visual-inertial odometry, sensor fusion, and motion-tracking applications.

## 1. Calibration Overview

The calibration workflow is divided into the following steps:

```bash
1. Record static IMU data
2. Estimate IMU noise parameters using Allan variance
3. Prepare imu.yaml for Kalibr
4. Calibrate camera intrinsic parameters
5. Record synchronized IMU-camera calibration data
6. Run Kalibr IMU-camera extrinsic calibration
7. Check and export the final calibration result
```

## 2. Repository Structure

```bash
.
├── config/
│   ├── imu.yaml              # IMU noise parameters for Kalibr
│   ├── camchain.yaml         # Camera intrinsic calibration result
│   └── aprilgrid.yaml        # AprilGrid calibration target configuration
│
├── bags/
│   ├── imu_static.bag        # Static IMU data for Allan variance analysis
│   └── imu_cam_calib.bag     # IMU-camera calibration rosbag
│
├── results/
│   ├── imu_allan_result/     # IMU noise calibration result
│   ├── camchain-imucam.yaml  # IMU-camera extrinsic calibration result
│   ├── report-imucam.pdf     # Kalibr report
│   └── results-imucam.txt    # Kalibr summary
│
└── README.md
```

## 3. IMU Calibration

Before running IMU-camera calibration, the IMU noise parameters need to be estimated.

A static IMU rosbag should be recorded first. During recording, the IMU must remain completely still.

Example:

```bash
rosbag record /imu0 -O imu_static.bag
```

The recording duration should be long enough for Allan variance analysis. A longer static recording usually gives more stable noise estimation results.

After collecting the static IMU data, use an Allan variance calibration tool to estimate the following parameters:

```yaml
accelerometer_noise_density
accelerometer_random_walk
gyroscope_noise_density
gyroscope_random_walk
```

These parameters are then written into `config/imu.yaml`.

Example `imu.yaml`:

```yaml
imu0:
  rostopic: /imu0
  update_rate: 400.0

  accelerometer_noise_density: 0.0014137774377636627
  accelerometer_random_walk: 4.209534659066097e-05

  gyroscope_noise_density: 9.6755597798026e-05
  gyroscope_random_walk: 3.449857059399396e-06
```

## 4. Camera Calibration

Camera intrinsic calibration should be completed before IMU-camera extrinsic calibration.

Example command:

```bash
kalibr_calibrate_cameras \
  --target config/aprilgrid.yaml \
  --bag bags/cam_calib.bag \
  --models pinhole-radtan \
  --topics /cam0/image_raw
```

The generated camera calibration file should be saved as:

```bash
config/camchain.yaml
```

## 5. IMU-Camera Calibration

After preparing the camera intrinsic file, IMU configuration file, AprilGrid target file, and synchronized IMU-camera rosbag, run:

```bash
kalibr_calibrate_imu_camera \
  --target config/aprilgrid.yaml \
  --cam config/camchain.yaml \
  --imu config/imu.yaml \
  --bag bags/imu_cam_calib.bag
```

Kalibr will estimate the spatial transformation between the IMU and camera frames.

## 6. Data Collection Requirements

For IMU noise calibration:

* Keep the IMU completely static.
* Avoid touching or moving the device during recording.
* Use a sufficiently long recording duration.

For IMU-camera extrinsic calibration:

* The IMU and camera must be rigidly fixed together.
* The AprilGrid should remain visible in the camera image.
* Move the device with sufficient rotation around different axes.
* Avoid motion blur.
* Avoid purely static or purely translational motion.
* Make sure the timestamps of IMU and camera messages are valid.

## 7. Output Files

The IMU noise calibration provides the parameters used in:

```bash
config/imu.yaml
```

The IMU-camera calibration generates files such as:

```bash
camchain-imucam.yaml
report-imucam.pdf
results-imucam.txt
```

The most important result is the extrinsic transformation between the IMU and camera.

In Kalibr, the result is usually represented as:

```yaml
T_cam_imu:
  - [r11, r12, r13, tx]
  - [r21, r22, r23, ty]
  - [r31, r32, r33, tz]
  - [0.0, 0.0, 0.0, 1.0]
```

This matrix represents the transformation from the IMU frame to the camera frame.

For downstream applications, make sure whether the required convention is `T_cam_imu` or its inverse `T_imu_cam`.

## 8. Topic Check

Before running calibration, check the rosbag topics:

```bash
rosbag info bags/imu_cam_calib.bag
```

Expected topics:

```bash
/imu0
/cam0/image_raw
```

The topic names must match the names written in `imu.yaml` and `camchain.yaml`.

## 9. Common Problems

### IMU topic not found

Check whether the IMU topic in `imu.yaml` matches the rosbag topic.

### Camera topic not found

Check whether the image topic in `camchain.yaml` matches the rosbag topic.

### Poor IMU noise calibration result

Possible reasons include:

* Static recording time is too short.
* IMU was moved during recording.
* IMU data frequency is unstable.
* The wrong IMU topic was used.

### Poor IMU-camera calibration result

Possible reasons include:

* Insufficient rotational excitation.
* AprilGrid detection is unstable.
* Motion blur exists in the image.
* Camera intrinsic calibration is inaccurate.
* IMU noise parameters are incorrect.
* IMU and camera timestamps are not synchronized.
* IMU and camera are not rigidly mounted.

## 10. Notes

This repository focuses on the complete calibration pipeline of an IMU-camera system, including both IMU noise calibration and IMU-camera extrinsic calibration.

The calibration results can be used for visual-inertial odometry, sensor fusion, motion tracking, and other applications that require accurate IMU-camera spatial alignment.
