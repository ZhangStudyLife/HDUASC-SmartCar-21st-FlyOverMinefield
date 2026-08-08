# Mode2/Mode4 WiFi 全链路遥测设计

日期：2026-08-09

## 1. 目标

在飞机端 `FC_Loop_500Hz()` 中，为 Mode2 和 Mode4 输出列位置完全一致的
VOFA JustFloat 日志，用于离线比较：

- Mode2：图像误差 -> 图像 PD -> 车速/转向角度前馈 -> 目标姿态 -> 实际姿态。
- Mode4：图像误差 -> 图像 PID 速度目标 -> 光流速度估计 -> 速度 PID -> 目标姿态 -> 实际姿态。
- 区分跟车滞后、停车超调来自控制结构、参数、车运动、图像质量、光流质量、姿态解耦还是速度估计。

本次只增加诊断遥测，不改变 Mode2、Mode4 的控制公式、参数、极性、调度频率或失效处理。

## 2. 已确认的约束

- `wifi_justfloat` 调用放在 `FC_Loop_500Hz()` 中。
- 仅当锁存模式为 Mode2 或 Mode4 时生成该日志。
- Mode2 和 Mode4 始终发送同一组 42 个用户通道，禁止因模式改变列位置。
- I1 直接发送模式号；模式专属字段在另一模式下发送 `0.0f`。
- `wifi_justfloat` 自动在 I0 添加毫秒时间戳，因此调用处不重复传时间戳。
- 后期可以从已发送量准确计算的量不额外占通道。
- Mode2 中的光流、光流速度估计只用于诊断，不参与 Mode2 控制。
- 空地串口不连接不是日志成立的前提；车数据是否有效由数据年龄和实际前馈量判断。
- 本阶段不修改 WiFi 协议层，也不增加新的日志开关或参数。

## 3. 采样与一致性

控制中间量由对应的 50 Hz 模式控制函数在实际计算时保存，500 Hz 日志只读取保存结果，
不在 `fc_loop.c` 中复制图像 PD、坐标旋转或前馈公式。这样可保证日志中的误差、P/D 项和
前馈项确实是最近一次控制更新所使用的值，而不是 500 Hz 时刻重新计算出的近似值。

光流原始积分、姿态解耦结果、有效标志和采样时间必须来自同一帧快照。不能直接记录已被
消费并清零的 `lc302_data.valid`，也不能让原始积分和解耦值来自不同帧。

每帧包含 43 个浮点数（I0 时间戳加 42 个用户通道）和 4 字节 JustFloat 帧尾，共 176 字节；
500 Hz 理论数据率为 88,000 B/s。当前链路由 `wifi_cmd` 通过 WiFi SPI 发送，不使用通用
WiFi-UART 的 115200 波特率。实机仍需通过 I0 连续性验证发送队列没有持续积压或丢帧。

## 4. 固定通道定义

| 通道 | 字段 | 单位 | 语义 |
| --- | --- | --- | --- |
| I0 | timestamp | ms | `wifi_justfloat` 自动添加的采样时间戳 |
| I1 | flight_mode | - | 锁存飞行模式，Mode2 为 2，Mode4 为 4 |
| I2 | lamp_valid | 0/1 | 与模式控制图像闭环相同的融合车灯有效标志 |
| I3 | tof_valid | 0/1 | 融合 TOF 原始有效标志 |
| I4 | car_data_age_ms | ms | 当前时刻减最近车数据更新时间；从未同步时为 -1 |
| I5 | image_err_x_px | px | 最近一次模式控制实际使用的 X 图像误差；图像闭环无效时为 0 |
| I6 | image_err_y_px | px | 最近一次模式控制实际使用的 Y 图像误差；图像闭环无效时为 0 |
| I7 | image_p_x | Mode2: deg；Mode4: cm/s | X 图像 PID 的实际 P 项 |
| I8 | image_d_x | Mode2: deg；Mode4: cm/s | X 图像 PID 的滤波后 D 项 |
| I9 | image_p_y | Mode2: deg；Mode4: cm/s | Y 图像 PID 的实际 P 项 |
| I10 | image_d_y | Mode2: deg；Mode4: cm/s | Y 图像 PID 的滤波后 D 项 |
| I11 | car_vel_x_mps | m/s | 车上传的 X 速度原值；过期时仍保留末值，由 I4 判断新鲜度 |
| I12 | car_vel_y_mps | m/s | 车上传的 Y 速度原值；过期时仍保留末值，由 I4 判断新鲜度 |
| I13 | car_yaw_deg | deg | 车上传的 Yaw 原值 |
| I14 | car_yaw_rate_dps | deg/s | 车上传的 Yaw 角速度原值 |
| I15 | mode2_car_angle_ff_x_deg | deg | Mode2 实际使用的 X 车速角度前馈；Mode4 填 0 |
| I16 | mode2_car_angle_ff_y_deg | deg | Mode2 实际使用的 Y 车速角度前馈；Mode4 填 0 |
| I17 | mode4_car_velocity_ff_x_cmps | cm/s | Mode4 叠加到 X 速度目标的车速前馈；Mode2 填 0 |
| I18 | mode4_car_velocity_ff_y_cmps | cm/s | Mode4 叠加到 Y 速度目标的车速前馈；Mode2 填 0 |
| I19 | turn_angle_ff_x_deg | deg | 当前模式实际使用的 X 转向加速度角度前馈 |
| I20 | turn_angle_ff_y_deg | deg | 当前模式实际使用的 Y 转向加速度角度前馈 |
| I21 | roll_target_deg | deg | 最终送入姿态角外环的 Roll 目标 |
| I22 | pitch_target_deg | deg | 最终送入姿态角外环的 Pitch 目标 |
| I23 | yaw_target_deg | deg | 最终送入姿态角外环的 Yaw 目标 |
| I24 | roll_actual_deg | deg | 飞机实际 Roll |
| I25 | pitch_actual_deg | deg | 飞机实际 Pitch |
| I26 | yaw_actual_deg | deg | 飞机实际 Yaw |
| I27 | height_mm | mm | 融合高度 |
| I28 | height_velocity_mps | m/s | 融合垂向速度 |
| I29 | mode4_velocity_target_x_cmps | cm/s | Mode4 最终 X 速度目标；Mode2 填 0 |
| I30 | mode4_velocity_target_y_cmps | cm/s | Mode4 最终 Y 速度目标；Mode2 填 0 |
| I31 | velocity_estimate_x_cmps | cm/s | Mode4 速度 PID 控制坐标中的 X 反馈，即 `-Pos_Est_vel_x_2` |
| I32 | velocity_estimate_y_cmps | cm/s | Mode4 速度 PID 控制坐标中的 Y 反馈，即 `-Pos_Est_vel_y_2` |
| I33 | mode4_velocity_pid_i_x_deg | deg | Mode4 X 速度 PID 的 I 项；Mode2 填 0 |
| I34 | mode4_velocity_pid_i_y_deg | deg | Mode4 Y 速度 PID 的 I 项；Mode2 填 0 |
| I35 | mode4_final_angle_ff_x_deg | deg | Mode4 限幅并低通后的最终 X 角度前馈；Mode2 填 0 |
| I36 | mode4_final_angle_ff_y_deg | deg | Mode4 限幅并低通后的最终 Y 角度前馈；Mode2 填 0 |
| I37 | lc302_raw_flow_x_integral | LC302 integral | 最近光流帧的 X 原始积分 |
| I38 | lc302_raw_flow_y_integral | LC302 integral | 最近光流帧的 Y 原始积分 |
| I39 | attitude_decoupled_flow_x | decoupled flow | 与 I37/I38 同帧的姿态解耦 X 光流 |
| I40 | attitude_decoupled_flow_y | decoupled flow | 与 I37/I38 同帧的姿态解耦 Y 光流 |
| I41 | flow_frame_valid | 0/1 | 最近一帧光流自身的有效标志，保持到下一帧 |
| I42 | flow_data_age_ms | ms | 当前时刻减最近光流帧时间；尚未收到帧时为 -1 |

Mode2 也发送 I31/I32 和 I37-I42，以便在完全相同的飞行条件下离线观察“若使用光流速度环会看到什么”；
这些量不得回接 Mode2 控制。

## 5. 模式专属填充规则

### Mode2

- I5-I10 使用 `g_mode2_imgx_pid`、`g_mode2_imgy_pid` 对应的最近一次实际输入和 P/D 状态。
- I15-I16、I19-I20 使用 Mode2 最近一次实际应用的前馈，车数据过期时应为 0。
- I17-I18、I29-I30、I33-I36 固定发送 0。
- I31-I32、I37-I42照常发送诊断值，但不影响控制。

### Mode4

- I5-I10 使用 `g_mode4_imgx_pid`、`g_mode4_imgy_pid` 对应的最近一次实际输入和 P/D 状态。
- I17-I20、I35-I36 使用 Mode4 最近一次实际应用的前馈，车数据过期时应按现有控制逻辑归零。
- I15-I16 固定发送 0。
- I29-I34 使用 Mode4 速度目标、控制坐标速度反馈和速度 PID 积分状态。

模式 Reset、图像/TOF 无效以及车数据超时时，新增的模式中间状态必须与现有控制结果同步清零，
不得把上一模式或上一有效周期的前馈误当作当前实际控制量。

## 6. 不发送但可离线计算的量

以下量不额外占通道：

```text
image_pd_x = image_p_x + image_d_x
image_pd_y = image_p_y + image_d_y

mode2_total_angle_correction_x = roll_target_deg - roll_mech_trim_deg
mode2_total_angle_correction_y = pitch_target_deg - pitch_mech_trim_deg

mode4_image_velocity_x = mode4_velocity_target_x_cmps - mode4_car_velocity_ff_x_cmps
mode4_image_velocity_y = mode4_velocity_target_y_cmps - mode4_car_velocity_ff_y_cmps

mode4_velocity_error_x = mode4_velocity_target_x_cmps - velocity_estimate_x_cmps
mode4_velocity_error_y = mode4_velocity_target_y_cmps - velocity_estimate_y_cmps

mode4_velocity_pid_total_x = roll_target_deg - roll_mech_trim_deg
                             - mode4_final_angle_ff_x_deg
mode4_velocity_pid_total_y = pitch_target_deg - pitch_mech_trim_deg
                             - mode4_final_angle_ff_y_deg

attitude_error_roll = roll_target_deg - roll_actual_deg
attitude_error_pitch = pitch_target_deg - pitch_actual_deg
```

光流观测速度沿用当前 `Pos_Est` 的换算：

```text
flow_velocity_x_cmps = height_mm * 0.001
                       * attitude_decoupled_flow_x * 0.48076923
flow_velocity_y_cmps = height_mm * 0.001
                       * attitude_decoupled_flow_y * 0.48076923
```

车的纵横向加速度、不同车速区间和转向阶段可由 I0、I11-I14 差分或坐标变换得到，
不重复发送加速度字段。

## 7. 代码边界

- `fc_mode2.c`：仅保存 Mode2 最近一次实际使用的图像误差、车速角度前馈和转向角度前馈；Reset 同步清零。
- `fc_mode4.c`：仅保存 Mode4 最近一次实际使用的图像误差、车速速度前馈、转向角度前馈和最终角度前馈；Reset 同步清零。
- `fc_mode.h`：只暴露 `fc_loop.c` 组帧必需的最小只读状态声明，不引入通用日志框架。
- `Pos_Est.c/.h`：提供最近一帧原始积分、解耦结果、有效标志和年龄的一致快照；不改变估计器计算。
- `fc_loop.c`：在 `FC_Loop_500Hz()` 内按固定 42 列组帧；不复制控制公式；其他模式不发送本日志。
- `wifi_justfloat`：不修改。

不清理现有无关注释日志，不重构相邻控制代码，不修改 IAR 工程设置。

## 8. 验证标准

静态验证：

1. 激活的统一日志调用只有一处，位于 `FC_Loop_500Hz()`。
2. 调用参数恰好为 42 个，未超过 60 个用户通道上限。
3. Mode2 和 Mode4 每个通道的字段位置完全相同。
4. Mode2 的 I17-I18、I29-I30、I33-I36 为 0；Mode4 的 I15-I16 为 0。
5. I37-I41 来自同一光流帧快照，I42 在未收到首帧时为 -1。
6. `git diff --check` 通过，无未完成描述。
7. 不从命令行调用 IAR；由用户完成工程编译。

实飞日志验证：

1. I1 能可靠区分 Mode2 和 Mode4，切换模式后列含义不变。
2. I0 在预期 2 ms 间隔附近连续；若出现持续跳变或缺帧，先处理链路吞吐，再做相位分析。
3. 图像无效时 I5-I10 清零；车数据过期时实际前馈清零且 I4 继续增长。
4. Mode4 的原始光流、解耦光流、速度估计和目标姿态可按时间对齐，足以分析光流延迟与噪声。
5. 可从日志重建 Mode2 图像 PD 到姿态目标链路，以及 Mode4 图像环、速度环到姿态目标链路。
