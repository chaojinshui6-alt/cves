# joint_trajectory_controller / RRBot unrealistic joint state reproduction (extreme finite joint position)

## Summary

This document describes a reproducible issue affecting `joint_trajectory_controller` in the `foxy` branch of `ros-controls/ros2_controllers`.

When the controller receives an **extreme but still finite joint position** command, it accepts the command and propagates it into joint state output. As a result, `/joint_states.position` may contain persistent unrealistic finite values.

A representative triggering input is:

- `joint_names = ['joint1', 'joint2']`
- `positions = [8.98846567431158e+307, -0.5]`
- `velocities = [0.0, 0.0]`
- `time_from_start = 2 s`

Under this input, `/joint_states.position[0]` becomes an unrealistic finite value instead of a valid physical joint state.

## Affected project

- Vendor: `ros-controls`
- Product: `ros2_controllers (joint_trajectory_controller)`
- Branch / version tested: `foxy`

Reproduction environment uses:

- `ros2_control_demos`
- RRBot demo

## Vulnerability type

This issue is best described as:

**Missing validation or saturation of extreme finite joint position commands leads to unrealistic joint state output**

This is a semantic correctness / robustness issue. The triggering input is not `NaN`/`INF`, but an extreme finite floating-point value.

## Test environment

- OS: Ubuntu 20.04
- ROS: ROS 2 Foxy
- Target demo: RRBot
- Launch file: `ros2_control_demo_bringup rrbot_system_position_only.launch.py`

## Related repositories

- `ros-controls/ros2_controllers`
- `ros-controls/ros2_control_demos`

## Root cause summary

From fuzzing and manual replay, the controller path accepts extreme but finite joint position values and allows them to influence the exported joint state.

Instead of rejecting the command, clamping it to a safe range, or safely degrading, the system continues publishing unrealistic finite values in `/joint_states.position`.

As a result, an extreme finite joint position command can be accepted and later corrupt joint state output, producing physically meaningless finite position fields.

## Reproduction steps

### 1. Start RRBot

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 launch ros2_control_demo_bringup rrbot_system_position_only.launch.py
```

### 2. Spawn the trajectory controller
```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 run controller_manager spawner.py joint_trajectory_position_controller
```

### 3. Confirm the controller is active
```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 control list_controllers
```

### 4. Publish an extreme but finite joint position command
```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash

ros2 topic pub --once /joint_trajectory_position_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory "
joint_names:
- joint1
- joint2
points:
- positions: [8.98846567431158e+307, -0.5]
  velocities: [0.0, 0.0]
  time_from_start: {sec: 2, nanosec: 0}
"
```

### 5. Observe joint state output

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/diffbot_ws/install/setup.bash
ros2 topic echo /joint_states
```

## Observed result

The joint state output includes unrealistic finite position values, for example:

```yaml
name:
- joint1
- joint2
position:
- 8.988465674311579e+307
- -0.49999999999999994
velocity:
- .nan
- .nan
effort:
- .nan
- .nan
```
In repeated observations, the affected joint may continue to publish extremely large finite values across multiple consecutive state updates. For example:
```yaml
name:
- joint1
- joint2
position:
- -6.72155345567264e+117
- -0.49999999999999994
velocity:
- .nan
- .nan
effort:
- .nan
- .nan
---
name:
- joint1
- joint2
position:
- -4.481035637115094e+117
- -0.49999999999999994
velocity:
- .nan
- .nan
effort:
- .nan
- .nan
---
name:
- joint1
- joint2
position:
- -2.9873570914100625e+117
- -0.49999999999999994
velocity:
- .nan
- .nan
effort:
- .nan
- .nan
```
## Expected result

The controller should handle such commands safely by doing one or more of the following:

- reject the command
- saturate or clamp the command to a safe range
- safely degrade without corrupting joint state output

It should not publish physically meaningless extreme finite values in /joint_states.position.

## Security impact

A system that relies on valid joint state output may fail when /joint_states.position contains unrealistic finite values. This can break downstream components that depend on meaningful robot joint state.

Potential impact includes:

- denial of service in higher-level control or motion logic
- invalid robot state feedback for downstream components
- incorrect kinematics or planning behavior
- unsafe downstream behavior if unrealistic joint state values are not handled correctly