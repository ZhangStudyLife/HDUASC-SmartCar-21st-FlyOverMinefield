# Car F Gyro-Z Control Migration Design

## Objective

Migrate the Loongson project's gyro-Z inner-loop architecture into
`CYT4BB7_Car_F/project/code/Controller/car_loop.c` while keeping the current
wheel-speed loop and CRSF speed command behavior intact.

The first version uses a fixed yaw-rate target of `0 dps`. It does not add the
camera/image outer loop yet.

## Control Chain

The 100 Hz control path will be:

```text
CRSF speed command
  -> base wheel speed
  -> gyro-Z incremental PI correction
  -> left/right wheel target allocation
  -> existing left/right speed loops
  -> motor PWM
```

`car_gyroz_control_100HZ()` will run after the base speed target is selected and
before either wheel-speed PID is updated.

## Units And Feedback Conversion

`s_car_yaw_rate_lpf_dps` is the filtered `gyro_z` measurement in degrees per
second. The Loongson controller converts gyro rate into an equivalent encoder
wheel-difference unit. The migration will preserve that conversion:

```text
equivalent_rate = -gyro_dps * (pi / 180) * 28.1448005
```

The fixed `0 dps` target converts to zero in the same unit. The controller error
is therefore:

```text
error = target_equivalent_rate - feedback_equivalent_rate
```

The leading negative sign is intentionally retained from the Loongson project.
Physical clockwise/counter-clockwise polarity must be verified on the vehicle
before enabling a nonzero yaw-rate target.

## Incremental PI Controller

The controller is implemented locally in `car_loop.c` with dedicated state. It
does not reuse `PID_Update()`, because the existing PID core is positional while
the Loongson gyro loop is incremental.

The Loongson loop runs at 200 Hz with `Kp=0.70` and `Ki=0.30`. The migrated loop
runs at 100 Hz, so the integral increment is doubled to preserve approximately
the same accumulation per second:

```text
Kp = 0.70
Ki = 0.60
Kd = 0.00
```

The discrete update is:

```text
delta_output = Kp * (error - last_error) + Ki * error
output = output + delta_output
last_error = error
```

Controller history and output are reset during `car_loop_init()`.

The first migration intentionally does not add a deadband, automatic zeroing,
new gain scheduling, or a separate output clamp. This keeps behavior directly
comparable with the active Loongson implementation. Existing motor PWM limits
remain the final actuator protection.

## Wheel Target Allocation

The Loongson `K_turn=2` one-sided slowdown mixer will be retained. Let
`base_speed` be the target selected from the CRSF command and `D` be the gyro
controller output:

```text
D > 0:
    left_target  = base_speed
    right_target = base_speed - 2 * D

D <= 0:
    left_target  = base_speed + 2 * D
    right_target = base_speed
```

The mixer does not use symmetric `base_speed +/- D`; it never increases the
outer wheel above the requested base target for forward driving.

## Public State

`car_loop.h` will declare `car_gyroz_control_100HZ()`. The controller's target,
converted feedback, error, and output will be exposed as volatile float globals
so they can be inspected with a debugger and added to telemetry later without
changing the control implementation.

The initial target is:

```text
g_car_gyroz_target_dps = 0.0
```

## Existing Behavior Preservation

The current uncommitted CRSF speed levels in `car_loop.c` are preserved:

```text
-700, -400, -200, 0, 200, 400, 700
```

The existing speed-loop gains, feedforward, encoder filtering, motor inversion,
and PWM limiting are outside this change.

## Verification

Static verification will confirm:

1. `car_gyroz_control_100HZ()` is called exactly once per 100 Hz speed update.
2. It runs after base target generation and before both speed PID calls.
3. At zero gyro feedback and reset controller state, both wheel targets equal
   the base speed.
4. Positive and negative controller outputs select the intended mixer branches.
5. Existing user modifications to the CRSF speed levels remain unchanged.

Hardware verification should first log gyro rate, converted feedback, controller
error, controller output, and both wheel targets with the driven wheels lifted.
If a manual clockwise rotation causes positive feedback instead of the expected
corrective response, only the gyro conversion sign should be changed; controller
gains should not be used to compensate for reversed polarity.
