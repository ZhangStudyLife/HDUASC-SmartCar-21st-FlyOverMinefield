# Mode2 直接图像 PD 姿态控制设计

## 目标

将飞机端 Mode2 最小化重写为直接图像闭环：融合车灯的像素误差经 PD 计算为 Roll/Pitch 角度修正，再叠加车速角度前馈和转向加速度角度前馈。新 Mode2 不使用光流速度反馈，Yaw 目标固定为 0 度。

本次只修改 Mode2 及其专属参数和 AirComm 注册；Mode3、Mode4、通用姿态环、图像融合和位置估计器不改。

## 控制数据流

Mode2 仍在 50 Hz 调用中完成水平控制：

```text
融合车灯像素误差
  -> 图像 PD 角度修正
  + 车速角度前馈
  + 转向加速度角度前馈
  -> 总修正限幅
  + 机械中值
  -> Roll/Pitch 目标
```

图像误差与 Mode4 保持一致：

```c
img_err_x = g_car_lamp_fused.cx - g_projection_center.cx;
img_err_y = g_car_lamp_fused.cy - g_projection_center.cy;
```

使用现有 `pid_t` 和 `PID_Update()` 生成角度修正：

```c
img_angle_x = PID_Update(&g_mode2_imgx_pid, 0.0f, -img_err_x, dt);
img_angle_y = PID_Update(&g_mode2_imgy_pid, 0.0f, -img_err_y, dt);
```

`PID_Init()` 的 Ki、Kff 和积分限幅固定为 0，不为这些固定值保留 Mode2 参数字段。不使用二次 P 项，因此不保留 Kp2 字段。

## 前馈与坐标

车端速度 `g_car_vel_x/g_car_vel_y` 单位为 m/s。延用 Mode4 的航向差旋转，将车体右向/前向速度转到飞机 Roll/Pitch 控制轴，再乘以单位为 `deg/(m/s)` 的前馈系数。默认 X/Y 均为 `1.0 deg/(m/s)`，使 1 m/s 车速只叠加 1 度姿态修正。

转向加速度前馈延用 Mode4 的 `omega x velocity` 计算、坐标旋转和 `atan(accel/g)` 角度换算，默认 X/Y 增益分别为 0.72/0.30。不再设置独立的转向前馈限幅，由最终总修正限幅统一约束。

只有车端数据时间戳有效且数据龄期小于 200 ms 时，才叠加车速和转向前馈；超时后两项同时置零。

## 角度合成与限幅

```c
roll_correction = img_angle_x + car_vel_ff_x + turn_ff_x;
pitch_correction = img_angle_y + car_vel_ff_y + turn_ff_y;

roll_correction = FC_Mode_Clamp(roll_correction,
                                -g_fc_params.mode2_angle_limit_deg,
                                 g_fc_params.mode2_angle_limit_deg);
pitch_correction = FC_Mode_Clamp(pitch_correction,
                                 -g_fc_params.mode2_angle_limit_deg,
                                  g_fc_params.mode2_angle_limit_deg);

roll_angle_target = FC_Mode_Clamp(roll_trim + roll_correction,
                                  -angle_target_max, angle_target_max);
pitch_angle_target = FC_Mode_Clamp(pitch_trim + pitch_correction,
                                   -angle_target_max, angle_target_max);
```

总修正默认限幅为 +/-10 度。限幅针对机械中值之外的修正量；叠加机械中值后仍保留全局姿态目标限幅作为最后保护。

## Mode2 参数

`fc_params_t` 只保留下列 Mode2 字段：

| 参数 | 默认值 | 单位 |
|---|---:|---|
| `mode2_img_x_kp` | 0.12 | deg/px |
| `mode2_img_x_kd` | 0.09 | deg/(px/s) |
| `mode2_img_x_d_lpf` | 3.0 | Hz |
| `mode2_img_y_kp` | 0.12 | deg/px |
| `mode2_img_y_kd` | 0.09 | deg/(px/s) |
| `mode2_img_y_d_lpf` | 3.0 | Hz |
| `mode2_car_vel_ff_x_deg_per_mps` | 1.0 | deg/(m/s) |
| `mode2_car_vel_ff_y_deg_per_mps` | 1.0 | deg/(m/s) |
| `mode2_turn_accel_ff_gain_x` | 0.72 | - |
| `mode2_turn_accel_ff_gain_y` | 0.30 | - |
| `mode2_angle_limit_deg` | 10.0 | deg |

AirComm 删除旧 Mode2 速度环、Ki、Kff 和积分限幅注册，仅注册上表字段。

## 状态和失效处理

Mode2 全局控制状态只保留 `g_mode2_imgx_pid` 和 `g_mode2_imgy_pid`。删除速度 PID、速度目标、光流反馈、旧前馈滤波残量、YawAlign 和 Yaw 扫描状态。

- 非飞行态：调用 `FC_Mode2_Reset()`，复位两个图像 PID，Roll/Pitch 回到机械中值，Yaw 目标置 0。
- 融合车灯或 ToF 高度无效：复位图像 PID，图像角度修正为 0；若车端数据仍新鲜，车速和转向前馈仍可使用。
- 车端数据超时：只清零车速和转向前馈，图像 PD 继续工作。
- `FC_Mode2_100Hz()`：仅将 `yaw_angle_target` 置为 0，不再维护额外状态。
- `FC_Mode2_Get_Fixed_Height_M()`：保持返回 1.1 m。

## 需要删除的旧内容

- `g_mode2_velx_pid/g_mode2_vely_pid`
- `g_mode2_velx_target/g_mode2_vely_target`
- `Pos_Est_vel_x/Pos_Est_vel_y` 和所有光流速度反馈
- 速度目标变化率和 Kff 滤波状态
- Mode2 YawAlign、Yaw 扫描函数及相关调试状态
- 厘米距离误差 `g_car_lamp_fused_distance_projectioncenter_2.x_cm/y_cm`
- 旧 Mode2 速度环和无用图像 PID 参数字段
- 只有注释而无实际用途的 Mode2 大块日志代码

## 验证标准

1. 静态引用检查：代码中不再存在被删除的 Mode2 速度 PID、目标和参数引用。
2. 方程检查：图像有效且所有前馈为 0 时，姿态修正等于 `0.12 * error + 0.09 * LPF3Hz(dError/dt)`。
3. 极性检查：右向车速产生正 Roll 前馈，前向车速产生负 Pitch 前馈。
4. 限幅检查：图像 PD 和全部前馈的合成修正不超过 +/-10 度。
5. 失效检查：图像/ToF 无效会复位 PD，车端数据超时会将车速和转向前馈清零，Yaw 目标始终为 0。
6. 编译验证由用户使用项目工程执行，不从命令行调用 IAR。
