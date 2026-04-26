# AutoCarROS2 / local_planner NaN path reproduction (finite `Path2D` with consecutive duplicate waypoint positions)

## Summary

This document describes a reproducible issue affecting the AutoCarROS2 goals-to-path pipeline.

When the planner receives a **finite but geometrically degenerate `Path2D` goal** containing **consecutive duplicate waypoint positions**, it may accept the goal and publish an invalid `/autocar/path` containing `NaN` values.

A representative triggering input is:

- `poses[0] = (0.0, 0.0, 0.0)`
- `poses[1] = (0.16153290475547288, 0.0, 0.0)`
- `poses[2] = (0.16153290475547288, 0.0, 0.0)`

Under this input, `/autocar/path` may contain persistent `.nan` values in `x`, `y`, and `theta`.

## Affected project

- Product: `AutoCarROS2`
- Affected pipeline: `/autocar/goals -> /local_planner -> /autocar/path`
- Message type: `autocar_msgs/msg/Path2D`

Reproduction environment uses:

- AutoCarROS2 workspace under `autocar_ws`
- ROS 2 Foxy
- Launch entry: `ros2 launch launches default_launch.py`

## Vulnerability type

This issue is best described as:

**Missing validation or safe handling of consecutive duplicate waypoint positions leads to NaN-valued path output**

This is a semantic correctness / robustness issue. The triggering input is not `NaN` or `INF`; all input fields are finite.

## Test environment

- OS: Ubuntu 20.04
- ROS: ROS 2 Foxy
- Target workspace: `autocar_ws`
- Launch file: `launches default_launch.py`

## Related component

- `AutoCarROS2`
- `local_planner`
- `/autocar/goals`
- `/autocar/path`

## Root cause summary

From fuzzing and manual replay, the planner path accepts finite `Path2D` goals whose adjacent waypoint positions are identical.

Instead of rejecting the goal, deduplicating adjacent points, skipping the zero-length segment, or safely degrading, the system continues and publishes `NaN` values in `/autocar/path`.

As a result, a finite but degenerate geometric input can corrupt published path output and produce physically meaningless path fields.

## Reproduction steps

### 1. Start AutoCarROS2

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash
ros2 launch launches default_launch.py
```

### 2. Confirm the relevant topics exist

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash
ros2 topic list | grep autocar
```

### 3. Publish a finite `Path2D` goal with consecutive duplicate waypoint positions

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash

ros2 topic pub --once /autocar/goals autocar_msgs/msg/Path2D "{poses: [{x: 0.0, y: 0.0, theta: 0.0}, {x: 0.16153290475547288, y: 0.0, theta: 0.0}, {x: 0.16153290475547288, y: 0.0, theta: 0.0}]}"
```

### 4. Observe path output

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash
ros2 topic echo /autocar/path
```

## Observed result

The published path contains `NaN` values, for example:

```yaml
poses:
- x: .nan
  y: .nan
  theta: .nan
- x: .nan
  y: .nan
  theta: .nan
```

In other reproduced observations, the affected path may continue to publish multiple consecutive `NaN` points, for example:

```yaml
poses:
- x: .nan
  y: .nan
  theta: .nan
- x: .nan
  y: .nan
  theta: .nan
- x: .nan
  y: .nan
  theta: .nan
- x: .nan
  y: .nan
  theta: .nan
---
```

## Additional fuzz-derived evidence

A confirmed fuzz-derived finite input that also triggered the issue was:

```yaml
poses:
- x: 0.0
  y: 0.0
  theta: -0.18159554662367938
- x: 0.16153290475547288
  y: 0.0
  theta: -3.1384642301543506
- x: 0.16153290475547288
  y: 0.0
  theta: -3.1384642301543506
```

All fields were finite. The corresponding recorded `/autocar/path` contained:

```yaml
[0] x=nan, y=nan, theta=nan
[1] x=nan, y=nan, theta=nan
```

Further fuzz-derived `NaN` cases were also observed under labels such as `reverse_heading`, `sharp_turn`, `theta_mismatch`, `long_segment`, and `small_perturb`, but extracted inputs showed that these cases still commonly contained the same underlying geometry pattern: two consecutive waypoint positions were identical.

## Expected result

The planner should handle such goals safely by doing one or more of the following:

- reject the goal
- remove duplicate adjacent waypoints
- skip zero-length segments
- safely degrade without corrupting path output

It should not publish `NaN` values in `/autocar/path`.

## Security impact

A system that relies on valid path output may fail when `/autocar/path` contains `NaN` values. This can break downstream components that depend on meaningful geometric path data.

Potential impact includes:

- denial of service in higher-level navigation or control logic
- invalid planner/controller handoff
- corrupted visualization and debugging output
- incorrect downstream behavior if `NaN` path fields are not handled safely
