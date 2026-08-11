# Mode2 Target-Speed Feedforward Design

## Scope

Reuse the existing car-to-air RUN_DATA packet without changing its 11-float
length. `data[5]` carries the car-body right target velocity and `data[6]`
carries the car-body forward speed-loop target. The timestamp remains at
`data[10]`.

The differential-drive car sends zero right target velocity and converts
`g_car_base_speed_target` to m/s for the forward target. The aircraft parses
both target velocities next to the existing actual velocities.

## Mode2 Control

For each car-body velocity axis, Mode2 low-pass filters
`target_velocity - actual_velocity` at 1 Hz and predicts acceleration with
asymmetric gains of 5.0 for acceleration and 3.9 for braking. The predicted
car-body acceleration is multiplied by 4.5 deg/(m/s^2), transformed into the
aircraft Roll/Pitch frame with the existing yaw-difference convention, and
added to the existing centripetal feedforward and image PD output.

The old actual-speed differentiation, 0.9 Hz acceleration filter, and 6.0
deg/(m/s^2) longitudinal gain are removed. The only final control limit
remains the existing per-axis target Euler-angle limit of +/-20 degrees.

## Verification

- Sender and receiver retain an 11-float packet and timestamp index 10.
- Zero target and actual velocity produce zero predicted acceleration.
- Positive forward target error produces the same Roll/Pitch polarity as the
  previous positive forward-acceleration path for every yaw difference.
- Offline replay shows immediate correctly signed launch/braking feedforward
  without the delayed encoder-differentiation spike.
