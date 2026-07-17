# Pos_Est2 Car-Speed Rejection Design

## Goal

Add a second Air-side body-frame velocity estimator without changing the existing
`Pos_Est` output or its consumers. The second estimator removes the translational
car-speed component measured by optical flow and publishes:

```c
void Pos_Est_Init_2(void);
void Pos_Est_Update_1000HZ_2(void);

extern float Pos_Est_vel_x_2;
extern float Pos_Est_vel_y_2;
```

The output contract remains X left-positive, Y forward-positive, in `cm/s`.

## Scope

The implementation changes only:

- `CYT4BB7_Air/project/code/Estimation/Pos_Est/Pos_Est.h`
- `CYT4BB7_Air/project/code/Estimation/Pos_Est/Pos_Est.c`
- `CYT4BB7_Air/project/user/main_cm7_0.c`

The old estimator remains active and continues publishing `Pos_Est_vel_x/y`.
No flight mode is switched to `Pos_Est_vel_x_2/y_2` in this change.

## Architecture

`Pos_Est_Update_1000HZ()` remains the sole owner of LC302 acquisition and gyro
decoupling. It publishes a private read-only snapshot containing:

- forward/right acceleration after the existing calibration and bias removal;
- aircraft yaw rate;
- accelerometer-bias lock state;
- decoupled LC302 X/Y values, sensor validity, frame time, and a frame sequence.

`Pos_Est_Update_1000HZ_2()` runs immediately after the old update. It never calls
an LC302 or `FlowGyroDecoupler_LC302` initialization/update function. It performs
1 kHz IMU prediction and consumes each shared optical-flow sequence exactly once.

Initialization and update order:

```text
Pos_Est_Init()
Pos_Est_Init_2()

IMU_Update_1000HZ()
Pos_Est_Update_1000HZ()
Pos_Est_Update_1000HZ_2()
```

`Pos_Est_Init_2()` only resets the second estimator and aligns its consumed frame
sequence. It does not initialize or reset sensor hardware.

## Observation Model

Car velocity input is right-positive/forward-positive in `m/s`. Let:

```c
yaw_diff = wrap(g_car_yaw - g_euler.yaw) * DEG_TO_RAD;

car_left = -100.0f * (g_car_vel_x * cosf(yaw_diff) +
                      g_car_vel_y * sinf(yaw_diff));
car_forward = 100.0f * (-g_car_vel_x * sinf(yaw_diff) +
                        g_car_vel_y * cosf(yaw_diff));
```

The decoupled LC302 observation is converted using the current fused height:

```c
flow_left = height_m * flow_dec_x * 0.48076923f;
flow_forward = height_m * flow_dec_y * 0.48076923f;
```

When car data is fresh and the fused car lamp is valid, the corrected observation
is:

```c
flow_observation_x = flow_left + 0.67f * car_left;
flow_observation_y = flow_forward + 0.67f * car_forward;
```

Otherwise the correction coefficient is zero and the second estimator still
fuses the ordinary optical-flow observation.

Car data is fresh only when `g_car_sync_time_ms > 0` and the source timestamp has
advanced within the last `200 ms`. A stale car velocity must never be applied.

## Fusion

The second estimator uses the same body-frame propagation convention as the old
estimator:

```c
velocity = R(-yaw_delta) * velocity;
velocity_x -= acceleration_right * 0.001f;
velocity_y += acceleration_forward * 0.001f;
```

For every new valid optical-flow frame:

```c
innovation = clamp(observation - prediction, -100, 100);
correction = 0.08f * innovation;
```

The correction vector is limited to `18 cm/s` per flow frame and the output
vector is limited to `250 cm/s`. A frame is rejected when the sensor or height is
invalid, the height is below `0.20 m` or above the existing TOF valid limit, a
required value is not finite, or the corrected observation magnitude exceeds
`300 cm/s`.

During optical-flow outages, the second estimator applies the same `150 ms` and
`500 ms` bounded inertial-hold decay policy as the existing estimator. When the
existing static accelerometer-bias learner is locked, the second velocity state
is reset to zero.

The first version intentionally does not duplicate the old multi-state soft
health/reacquisition machine. Sensor validity, bounded innovation, bounded
per-frame correction, observation rejection, outage decay, and output limiting
provide the required compact protection while avoiding a second large state
machine.

## Parameters

The initial fixed deployment parameters are:

```text
car translation coefficient: 0.67
optical-flow correction gain: 0.08
```

The car coefficient is the median of four held-out moving time blocks:
`0.6694`, `0.6594`, `0.6693`, and `0.6962`. The selected gain gives the best
conservative moving-error result while avoiding the faster/noisier initial
response of gains above `0.10`.

These constants are not added to `fc_params` or the communication menu in this
change.

## Offline Verification

The replay reads the 33-column `D:\Downloads\光流车速debug.csv` log. It must:

- identify optical-flow updates only from `I13 flow_new_frame`;
- never inject repeated `I17/I18` values on every 1 kHz telemetry row;
- use the same coordinate transforms, clamps, limits, and update order as C;
- keep fitting and evaluation in separate contiguous time blocks;
- use image-derived velocity only as a delayed scoring reference, never as an
  estimator input other than the runtime fused-lamp validity gate.

The prior causal held-out comparison is the acceptance baseline:

```text
moving RMSE: old 88.92 cm/s, target approximately 58.27 cm/s
turning RMSE: old 93.04 cm/s, target approximately 57.48 cm/s
```

Because no independent motion-capture or ground-truth velocity is logged, these
metrics demonstrate relative improvement against the image/car-speed reference;
they do not prove absolute velocity accuracy.

## Acceptance Criteria

1. The four new public symbols exist and the old four public symbols remain.
2. LC302 acquisition and gyro decoupling are performed exactly once per cycle.
3. Each LC302 frame is consumed at most once by the second estimator.
4. Car correction is disabled after `200 ms` of stale car data.
5. Existing `Pos_Est_vel_x/y` calculations and all current consumers remain
   unchanged.
6. The firmware replay matches the implemented equations and improves the held-
   out moving and turning metrics over the logged old estimator.
7. `git diff --check`, symbol/call-site searches, and structural balance checks
   pass. Per the Air project rules, no command-line IAR build is attempted.

## Non-Goals

- Switching Mode4 or another controller to the new velocity output.
- Adding tunable parameters or AirComm/car-menu entries.
- Modeling vehicle rotational surface velocity or aircraft yaw-flow leakage.
- Adding dynamic image-area or image-offset scaling of the car coefficient.
- Refactoring or cleaning adjacent legacy code.
