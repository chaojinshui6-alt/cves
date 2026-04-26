# AutoCarROS2 / local_planner singleton path reproduction (finite `Path2D` with near-zero adjacent waypoint segment)

## Summary

This document describes a reproducible issue affecting the AutoCarROS2 goals-to-path pipeline.

When the planner receives a **finite but geometrically degenerate `Path2D` goal** containing a **near-zero-length segment between adjacent waypoints**, it may accept the goal and publish an underspecified `/autocar/path` containing only a **single finite pose** instead of a normal multi-point path.

A representative triggering input is:

- `poses[0] = (-0.02399468011343231, 0.13947714739772565, -0.680725855233268)`
- `poses[1] = (-0.023995298746320188, 0.13947636878846945, 2.4608667983565247)`
- `poses[2] = (0.06319318685302414, 0.13947737348591185, 2.460866798356525)`

Under this input, `/autocar/path` may collapse to a singleton path such as:

- `poses_len = 1`
- `poses[0] = (-0.02399468011343231, 0.13947714739772565, -2.2422009807334558)`

## Affected project

- Product: `AutoCarROS2`
- Affected planning interface: finite goals on `/autocar/goals` leading to corrupted output on `/autocar/path`
- Targeted node in fuzzing setup: `/local_planner`
- Message type: `autocar_msgs/msg/Path2D`

Reproduction environment uses:

- AutoCarROS2 workspace under `autocar_ws`
- ROS 2 Foxy
- Launch entry: `ros2 launch launches default_launch.py`

## Vulnerability type

This issue is best described as:

**Improper handling of near-zero adjacent waypoint geometry leads to an underspecified singleton path output**

This is a semantic correctness / robustness issue. The triggering input is not `NaN` or `INF`; all input fields are finite.

## Test environment

- OS: Ubuntu 20.04
- ROS: ROS 2 Foxy
- Target workspace: `autocar_ws`
- Launch file: `launches default_launch.py`

## Related component

- `AutoCarROS2`
- `/local_planner`
- `/autocar/goals`
- `/autocar/path`

## Root cause summary

From fuzzing and manual replay, the planner path accepts finite `Path2D` goals whose adjacent waypoints form a near-zero-length segment.

Instead of rejecting the goal, merging nearly identical points, skipping the degenerate segment, or safely degrading, the system continues and publishes a path containing only one finite pose.

As a result, a finite but degenerate geometric input can collapse the planner output into an underspecified path that is unlikely to satisfy downstream expectations for a normal trackable multi-point path.

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

### 3. Publish a finite `Path2D` goal with a near-zero adjacent waypoint segment

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash

ros2 topic pub --once /autocar/goals autocar_msgs/msg/Path2D "{poses: [{x: -0.02399468011343231, y: 0.13947714739772565, theta: -0.680725855233268}, {x: -0.023995298746320188, y: 0.13947636878846945, theta: 2.4608667983565247}, {x: 0.06319318685302414, y: 0.13947737348591185, theta: 2.460866798356525}]}"
```

### 4. Observe path output

```bash
source /opt/ros/foxy/setup.bash
source ~/robolocal/robofuzz/targets/autocar_ws/install/setup.bash
ros2 topic echo /autocar/path
```

## Observed result

The published path degenerates to a singleton finite path, for example:

```yaml
poses:
- x: -0.02399468011343231
  y: 0.13947714739772565
  theta: -2.2422009807334558
```

In the corresponding fuzz-derived rosbag, the same case produced:

- `/autocar/path count = 1`
- `poses_len = 1`
- `poses[0] = (-0.02399468011343231, 0.13947714739772565, -2.2422009807334558)`

while `/autocar/cmd_vel` and `/autocar/odom` were still active.

## Expected result

The planner should handle such goals safely by doing one or more of the following:

- reject the goal
- merge or remove nearly identical adjacent waypoints
- skip the near-zero-length segment
- safely degrade without collapsing path output to a singleton path

It should not publish an underspecified single-pose `/autocar/path` for such finite input if downstream components expect a normal multi-point path.

## Security impact

A system that relies on valid path structure may fail when `/autocar/path` collapses to a singleton path. This can break downstream components that assume a normal multi-point trajectory.

Potential impact includes:

- denial of service in higher-level navigation or tracking logic
- invalid planner/controller handoff
- incorrect path-following behavior
- unsafe downstream behavior if singleton path output is not handled correctly

## Suggested fixes

1. Validate incoming goals for near-zero adjacent waypoint segments.
2. Merge or remove nearly identical adjacent points before path generation.
3. Enforce minimum path structure before publishing `/autocar/path`.
4. Fail closed when degenerate geometry prevents generation of a proper multi-point path.
5. Add regression tests for near-zero adjacent waypoint inputs.

## Proposed title for an issue report

**AutoCarROS2 collapses finite degenerate goals into singleton `/autocar/path` output under near-zero adjacent waypoint geometry**
