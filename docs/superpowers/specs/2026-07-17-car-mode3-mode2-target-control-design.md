# 车端 Mode3 复制 Mode2 飞机目标控制设计

## 目标

- 车端 Mode3 不再使用遥控器 CH0/CH1 生成速度目标。
- Mode3 像当前 Mode2 一样，接收飞机下发的前进速度、横移速度和 yaw 目标，驱动车辆执行灭灯规划。
- Mode3 保留独立的状态、PID 和 `mode3_velocity_*` 参数，后续可与 Mode2 分别调参。

## 控制数据链

- 速度目标读取 `g_air_car_plan_forward_mps` 和 `g_air_car_plan_strafe_mps`。
- `g_air_car_plan_valid > 0.5f` 时使用飞机速度目标，否则将前进和横移目标清零。
- 两轴速度目标均按当前 Mode2 的 `1.5 m/s` 最大值限幅。
- 目标低通、里程计反馈、速度前馈、两轴 PID、输出限幅和横移符号全部复制当前 `car_mode2.c`。
- Mode3 使用自己的 `g_car_mode3_state`、两个 PID 实例和 10 个 `mode3_velocity_*` 参数。
- `car_loop.c` 将 `CAR_MODE_3` 加入飞机 yaw 目标模式列表，使用 `g_air_yaw_angle_target_deg` 转为弧度后传入 `Control_100Hz()`。
- AirComm 运行数据超时时仍由现有上层逻辑禁用车辆控制，不在 Mode3 内增加重复超时判断。

## 修改范围

- `CYT4bb7_Car/project/code/Controller/car_mode3.c`
  - 用当前 Mode2 算法替换当前基于 Mode4 的遥控器算法。
  - 仅将 Mode2 标识符和参数前缀映射为 Mode3。
- `CYT4bb7_Car/project/code/Controller/car_loop.c`
  - 在 yaw 目标选择条件中加入 `CAR_MODE_3`。

## 保持不变

- 不修改飞机端代码和 AirComm 协议字段。
- 不修改参数表数量、参数顺序、默认值、范围或菜单结构。
- 不修改 Mode2、Mode4、Mode8 和底层 `Control_100Hz()`。
- 不抽取 Mode2/Mode3 公共函数，不共享运行状态。

## 验证标准

- Mode3 不再引用 `g_air_crsf_std_ch0`、`g_air_crsf_std_ch1` 或 `Control_GetYawAngle()`。
- Mode3 引用飞机规划有效标志及前进、横移速度目标。
- 将 Mode2/Mode3 标识符和参数前缀归一化并忽略注释、空行后，可执行代码无差异。
- `car_loop.c` 对 Mode3 使用飞机下发 yaw 目标。
- `git diff --check` 通过。
- 车端 CM7_0 IAR 构建通过。

