# Mode8 搬运 Mode2 运动前馈设计

## 目标

在 Air 侧 `fc_mode8.c` 中独立实现 Mode2 当前使用的车速 yaw 坐标系转换和转弯向心加速度前馈。Mode8 保留自己的参数名和状态，不直接引用 Mode2 控制状态。

## 参数

`fc_params.c` 中 Mode8 的图像 PID、速度 PID、速度 KFF 和 `kp_car_x/y` 默认值已经与 Mode2 完全一致，因此不修改参数表。

Mode8 新增两个独立全局参数，当前数值复制自 Mode2：

- `g_mode8_turn_accel_ff_gain = 0.6f`
- `g_mode8_turn_accel_ff_limit_deg = 16.0f`

## 控制逻辑

只修改 `CYT4BB7_Air/project/code/FlightController/fc_mode8.c`：

1. 接入 `g_car_yaw`、`g_car_yaw_rate_dps`、`g_car_sync_time_ms`、`g_car_last_update_time_ms` 和本机毫秒计数。
2. 使用 `g_car_yaw - g_euler.yaw` 将车体横向/前向速度旋转到飞机 roll/pitch 控制坐标系。
3. 使用车体 yaw 角速度与车速计算 `omega x velocity` 转弯加速度，再旋转到飞机坐标系并换算为角度前馈。
4. 向心前馈使用 Mode8 独立的 `0.6f` 增益和单轴 `16.0f` 限幅。
5. 车端时间戳无效或超过 200 ms 时，不叠加车速前馈，并立即清零 Mode8 角度 KFF 低通状态。
6. 保留 Mode8 的 `img_err_compare()`、融合图像反馈、模式入口和 yaw 对准逻辑。

## 不修改

- 不修改 Mode2。
- 不抽取公共函数。
- 不修改 Air/Car 通信协议。
- 不修改 Mode8 遥测内容。
- 不处理现有无关警告或死代码。

## 验证

- 对照 Mode2 逐项检查旋转公式、符号、常量、限幅和断链行为。
- 搜索确认新增外部量都有真实定义。
- 运行 `git diff --check`。
- 按 Air 侧 `AGENTS.md` 约束，不从命令行调用 IAR，由用户执行实际工程编译。
