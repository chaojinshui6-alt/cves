# Finding: AutoCarROS2 publishes `NaN` path for finite goals with consecutive duplicate waypoints

## Summary

AutoCarROS2 can publish an invalid `/autocar/path` containing `NaN` values when it receives a finite `autocar_msgs/msg/Path2D` goal that contains **consecutive duplicate waypoint positions**.

This issue was:

* observed during fuzzing,
* confirmed by extracting the concrete finite input from the fuzz queue,
* and manually reproduced with `ros2 topic pub` + `ros2 topic echo`.

This is not a timeout or observation-window artifact. The system actually publishes `NaN` on `/autocar/path`.

---

## Impact

Publishing `NaN` in `/autocar/path` is a correctness and robustness failure.

Potential impact:

* downstream controllers may consume invalid path data,
* visualization and debugging tools receive corrupted path output,
* the planner may enter a persistently bad state and continue publishing invalid paths,
* the system does not degrade gracefully on a realistic degenerate input.

---

## Confirmed trigger condition

The currently confirmed bug family is:

> **finite `Path2D` input with consecutive duplicate waypoint positions**

This means either:

* `poses[0].x/y == poses[1].x/y`, or
* `poses[1].x/y == poses[2].x/y`

In geometric terms, this creates a **zero-length segment** between two adjacent waypoints.

---

## Minimal manually reproduced input

The following input reproduces the problem:

```yaml
poses:
- x: 0.0
  y: 0.0
  theta: 0.0
- x: 0.16153290475547288
  y: 0.0
  theta: 0.0
- x: 0.16153290475547288
  y: 0.0
  theta: 0.0
```

Equivalent command:

```bash
ros2 topic pub --once /autocar/goals autocar_msgs/msg/Path2D "{poses: [{x: 0.0, y: 0.0, theta: 0.0}, {x: 0.16153290475547288, y: 0.0, theta: 0.0}, {x: 0.16153290475547288, y: 0.0, theta: 0.0}]}"
```

Observed output:

```yaml
poses:
- x: .nan
  y: .nan
  theta: .nan
- x: .nan
  y: .nan
  theta: .nan
```

---

## Fuzzing evidence

One confirmed fuzz-derived case used the following finite input:

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

All fields were finite.

The corresponding recorded `/autocar/path` contained:

```yaml
[0] x=nan, y=nan, theta=nan
[1] x=nan, y=nan, theta=nan
```

---

## What has been ruled out

The following are **not required** for reproduction:

* unusual `theta` values,
* very small coordinate magnitude,
* fuzz-only execution.

Manual reproduction showed that **duplicate adjacent waypoint positions alone** are sufficient.

---

## Important note about other fuzz modes

During fuzzing, `NaN` was also observed under labels such as:

* `reverse_heading`
* `sharp_turn`
* `theta_mismatch`
* `long_segment`
* `small_perturb`
* `generic_fallback`

However, extracting the concrete queued inputs showed that these cases still commonly contained the same underlying geometry pattern:

> **two consecutive waypoint positions were identical**

So at this point, the only **confirmed independent trigger family** is the duplicate-consecutive-waypoint case.

---

## Why this matters in practice

Consecutive duplicate waypoints are realistic. They can arise from:

* upstream planner or waypoint generator deduplication bugs,
* repeated GUI/RViz clicks on the same location,
* quantization or rounding,
* path simplification / smoothing edge cases,
* localization or state-machine stalls that emit the same point twice.

A robust planner should reject, sanitize, or gracefully handle such inputs instead of publishing `NaN`.

---

## Likely root cause

A likely cause is missing protection around **zero-length path segments**, for example:

* normalizing a zero vector,
* dividing by segment length when length is zero,
* undefined tangent / heading / curvature computation,
* failure to validate intermediate results before publishing `/autocar/path`.

---

## Expected behavior

When given consecutive duplicate waypoints, the planner should do one of the following:

* reject the input,
* remove duplicate adjacent points,
* skip zero-length segments,
* publish a finite degraded path,
* or explicitly return failure without publishing invalid path data.

It should **not** publish `NaN` values on `/autocar/path`.

---

## Suggested fixes

1. Validate incoming goals for consecutive duplicate waypoint positions.
2. Remove or skip zero-length segments before path generation.
3. Add finite-value validation before publishing `/autocar/path`.
4. Fail closed when internal geometry computation becomes invalid.
5. Add regression tests for duplicate-adjacent-waypoint inputs.

---

## Proposed title for an issue report

**AutoCarROS2 publishes NaN-valued `/autocar/path` for finite goals with consecutive duplicate waypoints**
