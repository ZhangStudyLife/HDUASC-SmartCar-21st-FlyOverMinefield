`git clone --branch national-2026 --depth 1 --recurse-submodules --shallow-submodules https://github.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield.git`

# HDUASC SmartCar 21st – 飞跃雷区

空地协同智能车竞赛项目。系统由 **无人机（飞机）** 与 **地面小车** 组成，无人机通过摄像头识别地面信标和车灯，将感知结果与自身姿态下发给小车，小车据此完成循迹导航。

```
┌─────────────────┐      SPI (SCB0)       ┌──────────────────────┐     UART AirComm     ┌──────────────────┐
│  CYT2BL3_Image   │ ◄──────────────────► │    CYT4BB7_Air        │ ◄──────────────────► │   CYT4bb7_Car     │
│  (空中感知层)     │                      │    (空中控制与通信中枢)  │                      │   (地面执行层)    │
│                  │  beacon(x,y,radius)  │                       │ 飞机姿态/高度/CRSF遥控 │                  │
│  MT9V03X 摄像头   │  ×4 + car_lamp ×1   │  双核 M7:             │ ←─────────────────── │  双核 M7:         │
│  连通域+PCA 特征   │ ──────────────────► │  ·CM7_0 飞控 1000Hz   │  车里程计速度          │  ·麦轮串级PID      │
│  多阈值级联检测    │                     │  ·CM7_1 SPI+图像转发   │ ────────────────────► │  ·编码器+IMU       │
│                  │  flight_state/       │                       │                      │  ·里程计+打滑检测  │
│  CYT2BL3 M4 单核  │ ◄────────────────── │  级联PID+Betaflight   │                      │                  │
└─────────────────┘    board_id            │  4×TOF高度融合        │                      └──────────────────┘
                                           │  Mahony AHRS          │
                                           └──────────────────────┘
```

## 项目结构

| 子仓库                             | 芯片                        | 职责                                                        |
| :--------------------------------- | :-------------------------- | :---------------------------------------------------------- |
| [`CYT2BL3_Image`](./CYT2BL3_Image/) | CYT2BL3 (Cortex-M4, 160MHz) | 空中感知：摄像头采集 + 信标/车灯识别，通过 SPI 上报飞机     |
| [`CYT4BB7_Air`](./CYT4BB7_Air/)     | CYT4BB7 (双核 Cortex-M7)    | 飞控核心：姿态解算、级联 PID 控制、与图像板和车板的双向通信 |
| [`CYT4bb7_Car`](./CYT4bb7_Car/)     | CYT4BB7 (双核 Cortex-M7)    | 地面执行：麦轮控制、里程计融合、循迹导航                    |

## 三子系统协作流程

```
1. 飞机摄像头(MT9V03X)采集 188×120 灰度图
        ↓
2. CYT2BL3 运行图像算法（BFS 连通域 + PCA 特征提取 + 多阈值级联）
        ↓ SPI 上行: 4个信标坐标 + 1个车灯矩形(位置/长宽/角度)
3. CYT4BB7_Air CM7_1 接收 → IPC → CM7_0 飞控算法
        ↓ AirComm UART 下行: 飞机姿态(roll/pitch/yaw) + 高度 + CRSF遥控通道
4. CYT4bb7_Car 解析控制指令 → 麦轮速度环PID → 电机PWM
        ↓ AirComm UART 上行: 车里程计速度
5. CYT4BB7_Air 接收车速度 → 跟车模式速度闭环
```

## 通信拓扑

### SPI: 图像板 ↔ 飞机

- **物理**: CYT4BB7 的 SCB6 (SPI Master) ↔ CYT2BL3 的 SCB0 (SPI Slave)，Motorola Mode 0
- **帧格式**: `AA 55` + CMD(0x20) + payload_len + payload + CRC16-LE + `ED`
- **上行** (2BL3→Air): 协议版本 + 4 beacon(各13B) + 1 car_lamp(21B) = **77B** 应用数据
- **下行** (Air→2BL3): magic(0x5A) + board_id + flight_state + 图传/显示使能 = **9B**

### AirComm UART: 飞机 ↔ 小车

- **物理**: UART3 @ 1.152Mbps
- **帧格式**: `AA AA 55 55` + payload + CRC16-CCITT, 100Hz 双向
- **飞机→车** (15 float): ToF高度 | roll/pitch/yaw | 水平速度 | 飞机状态 | CRSF 8通道
- **车→飞机** (10 float): 车里程计速度 vel[x]/vel[y] | 预留字段
- **心跳**: 200ms 间隔，600ms 未收到判定离线

## 关键技术点

### 图像板 (`CYT2BL3_Image`)

- **多阈值级联**: 4 级分割 (200→150→130→100)，逐级检测不同亮度的信标
- **车灯掩膜**: 先检测车灯（高阈值 200 + elongation > 1.6），构建 mask 后再检测信标，避免灯周围亮像素干扰
- **PCA 特征提取**: 对每个连通域计算协方差矩阵→特征值分解→长轴/短轴/倾角/等效半径
- **帧间跟踪**: 最近邻匹配（距离阈值 36px），tracked slot 限制 4 个，灯出现后 120 帧允许新 slot
- **双板支持**: board_id=0 和 board_id=1 各运行独立算法实例
- **WiFi 图传**: ESP32 SPI 模块 TCP 推流到上位机调试

### 飞机 (`CYT4BB7_Air`)

- **双核分工**: CM7_0 跑飞控 1000Hz 实时任务，CM7_1 管理 2 块 2BL3 SPI 通信 + IPC 转发
- **级联 PID**: 位置(50Hz)→速度(100Hz)→角度(500Hz)→角速度(1000Hz)→混控
- **增强 PID**: PT3 前馈平滑、PT3 输出低通、二阶 Butterworth D 项滤波、Anti-Windup、积分松弛
- **高度融合**: 4×VL53L1X TOF（机臂下方各一）+ 加速度计惯性导航
- **悬停油门在线学习**: 借鉴 ArduPilot MOT_THST_HOVER
- **9 种飞行模式**: MODE_4(跟图像信标) / MODE_5(跟车) / MODE_7(跟车V2)

### 小车 (`CYT4bb7_Car`)

- **麦轮控制**: 逆运动学解算 + 串级 PID(yaw角度→角速度→轮速) + 四轮独立前馈(KS/KV/KSTART)
- **里程计融合**: 麦轮正运动学 + 打滑检测(门控滤波) + 信标位置修正(fixator)
- **传感器**: ICM42688 IMU(Mahony AHRS) + 四路正交编码器

## 仓库初始化

```bash
# 首次克隆（含子模块）
git clone --recursive <repo-url>

# 已克隆后补齐子模块
git submodule update --init --recursive

# 拉取各子模块远端最新
git submodule update --init --recursive --remote
```

## 联调文档

- 空地通信协议: [`空地通信互传数据手册.md`](./空地通信互传数据手册.md)
- 飞控与图像板 SPI 设计: [`CYT4BB7_Air/doc/飞控和图像板的SPI主从通信设计方案.md`](./CYT4BB7_Air/doc/飞控和图像板的SPI主从通信设计方案.md)
- 车端摄像头迁移: [`CYT4bb7_Car/docs/air_camera_comm_migration.md`](./CYT4bb7_Car/docs/air_camera_comm_migration.md)

## 版本管理

- 总仓库通过 `git submodule` 固定三个子项目的提交版本
- 联调以总仓库记录的指针为准，避免版本漂移
- 更新子项目后需在总仓库提交新的 submodule 指针
