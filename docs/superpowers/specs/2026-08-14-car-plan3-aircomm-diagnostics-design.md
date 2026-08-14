# CarPlan3 AirComm Diagnostic Mapping

## Goal

Keep the existing AirComm packet lengths and speed-field indexes while replacing
the obsolete car_plan diagnostic values with the corresponding car_plan3 result.

## Data Contract

The 52-float diagnostic packet keeps all existing indexes except `20..22`:

| Index | New value | Unit |
| --- | --- | --- |
| `20` | `car_plan_3_result_t.camera_mask` | bit mask |
| `21` | `car_plan_3_result_t.target_center_x` | m |
| `22` | `car_plan_3_result_t.target_center_y` | m |

The critical 15-float packet is unchanged. Its plan validity and velocity fields
continue to come from car_plan3.

## Air Side

`send_air_run_data_200hz()` writes the three car_plan3 values directly into
diagnostic indexes `20..22`. car_plan and car_plan2 continue to update, but none
of their result fields enter the AirComm packet.

## Car Side

The receiver stores indexes `20..22` in variables named for their new meanings:

- `g_air_car_plan_camera_mask`
- `g_air_car_plan_target_center_x_m`
- `g_air_car_plan_target_center_y_m`

The status menu displays the camera mask instead of the obsolete single-camera
number. No control logic consumes these three diagnostic fields.

## Compatibility

This is a semantic breaking change for diagnostic consumers of indexes `20..22`.
Packet length and all control-critical indexes remain unchanged. The airplane and
Car_F firmware must therefore be updated together.

## Verification

- Search confirms indexes `20..22` use only car_plan3 fields.
- Search confirms obsolete car-side variable names are removed.
- `git diff --check` passes in each affected repository.
- Air CM7_0 and Car_F CM7_0 Debug builds complete with zero errors.
