\# diff\_drive\_controller / DiffBot NaN odometry reproduction (extreme angular velocity)



\## Summary



This document describes a reproducible issue affecting `diff\_drive\_controller` in the `foxy` branch of `ros-controls/ros2\_controllers`.



When the controller receives an \*\*extreme but still finite angular velocity\*\* command, it accepts the command and propagates it into odometry computation. As a result, `/odom` may contain `NaN` position and orientation values.



A representative triggering input is:



\- `linear.x = 0.5`

\- `angular.z = 8.98846567431158e+307`



Under this input, `/odom.pose.pose.position.x`, `/odom.pose.pose.position.y`, and the odometry orientation quaternion become `NaN`.



\## Affected project



\- Vendor: `ros-controls`

\- Product: `ros2\_controllers (diff\_drive\_controller)`

\- Branch / version tested: `foxy`



Reproduction environment uses:



\- `ros2\_control\_demos`

\- DiffBot demo



\## Vulnerability type



This issue is best described as:



\*\*Missing validation or saturation of extreme angular velocity commands leads to NaN odometry\*\*



This is a semantic correctness / robustness issue. The triggering input is not NaN/INF, but an extreme finite floating-point value.



\## Test environment



\- OS: Ubuntu 20.04

\- ROS: ROS 2 Foxy

\- Target demo: DiffBot

\- Launch file: `ros2\_control\_demo\_bringup diffbot\_system.launch.py`



\## Related repositories



\- `ros-controls/ros2\_controllers`

\- `ros-controls/ros2\_control\_demos`



\## Root cause summary



From source inspection, the controller checks whether incoming command values are finite, but it does not sufficiently reject or saturate extreme finite floating-point values before they influence the control and odometry path.



As a result, an extreme angular velocity command can be accepted and later corrupt odometry output, producing `NaN` position and orientation fields.



\## Reproduction steps



\### 1. Start DiffBot



```bash

source /opt/ros/foxy/setup.bash

source \~/robolocal/robofuzz/targets/diffbot\_ws/install/setup.bash

ros2 launch ros2\_control\_demo\_bringup diffbot\_system.launch.py

