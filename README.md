# 第 21 届全国大学生智能汽车竞赛：飞跃雷区

> [比赛年份] 年，[学校 / 队伍名称]参加第 21 届全国大学生智能汽车竞赛“飞跃雷区”组，获得全国总决赛冠军。这里是整套项目的母仓库，代码、PCB、结构件、调试工具和文档都从这里开始找。

比赛成绩、队伍名称、现场照片和最终成绩单由我后续补充。现在先把项目的阅读入口和仓库关系整理清楚，免得别人一上来就掉进某个子仓库的代码里。

## 先看这里

如果只想先了解我们做了什么，建议按这个顺序：

1. [待填写：项目 / 比赛演示视频]
2. [国赛结构讲解视频](https://www.bilibili.com/video/BV1894o62Ej4/)
3. [上位机调试路径规划和图像兜底](https://www.bilibili.com/video/BV1Rm4m6fEMv/)
4. [Air 飞控最新总文档](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/README.md)
5. 再根据下面的模块入口，跳到自己真正关心的部分。

这几个视频和总文档是整个开源内容的“入口层”。具体实现会分散在不同子仓库里，但不建议把它们当成第一阅读材料。

## 我们做的是什么

这是一个空地协同的智能车项目：无人机在空中识别地面信标和车灯，把视觉结果、飞机姿态和高度等信息交给飞控；飞控经过相机模型、三摄融合和路径规划，计算车模应该行驶的方向和速度；车模再执行速度控制，并把实际速度回传给飞机。

简单说就是：

```text
空中图像板识别信标 / 车灯
        ↓ SPI
CYT4BB7_Air：双核飞控、姿态高度、相机模型、CarPlan3
        ↓ AirComm UART
CYT4bb7_Car：麦轮控制、编码器、里程计和车模执行
```

## 整体方案

项目的主要数据流如下：

```text
MT9V03X 摄像头
    -> CYT2BL3_Image 识别信标和车灯
    -> Camera SPI
    -> CYT4BB7_Air CM7_1 接收并通过 IPC 发布
    -> CM7_0 读取 image_data
    -> Three_Camera 将像素反投影到水平坐标
    -> CarPlan3 过滤、融合、选目标并输出车模速度
    -> AirComm 下发给 CYT4bb7_Car
    -> 车模执行并回传实际速度
```

这里面每一层都有自己的边界：图像板负责“看见了什么”，Air 负责“这些目标在空间中在哪里、车应该往哪里走”，Car 负责“车轮到底怎么转”。后面的文档也按这个边界组织，遇到问题时可以先判断它属于哪一层。

## 核心入口

总仓库固定的子模块版本和子仓库远端最新文档不是一回事，所以这里明确分成两列：

| 主题     | 子仓库 / 文档最新入口                                                                                                                                                                      | 当前总仓库固定版本                                       |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| 国赛结构 | [结构总文档](./structure/README.md)                                                                                                                                                           | [当前国赛结构文件](./structure/national/)                 |
| 省赛结构 | [结构总文档](./structure/README.md)                                                                                                                                                           | [当前省赛结构文件](./structure/provincial/)               |
| 硬件 PCB | [硬件 PCB 总文档](./hardware/README.md)                                                                                                                                                     | [当前 PCB 源文件](./hardware/boards/)                     |
| Air 飞控 | [Air 最新总文档](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/README.md)                                                                                                | [固定版本 Air](./CYT4BB7_Air/)                            |
| 图像板   | [Image 最新仓库](https://github.com/ZhangStudyLife/CYT2BL3_Image/tree/national-2026)                                                                                                        | [固定版本 Image](./CYT2BL3_Image/)                        |
| 上位机   | [上位机最新主文档](https://github.com/ZhangStudyLife/BeaconImageAnalyzer/blob/main/README.md)；[CarPlan 分支](https://github.com/ZhangStudyLife/BeaconImageAnalyzer/blob/car_plan/README.md) | [固定版本上位机](./BeaconImageAnalyzer/)                  |
| 车端     | [Car 仓库入口](https://github.com/choumouing/CYT4bb7_Car/)；[Car_F 国赛分支](https://github.com/ZhangStudyLife/CYT4BB7_Car_F/tree/national-2026)                                             | [固定版本 Car](./CYT4bb7_Car/) / [Car_F](./CYT4BB7_Car_F/) |

Car 端的具体文档目前不是这套开源内容的主线，先把它作为完整工程入口保留。真正想理解“飞机为什么能让车跑起来”，建议先看 Air、硬件和结构，再回头看 Car。

## 按目标阅读

### 只想了解比赛方案

先看本页的[整体方案](#整体方案)，然后看[结构总文档](./structure/README.md)和[硬件 PCB](./hardware/README.md)。结构总文档会直接引流国赛/省赛视频和对应的 SolidWorks 文件，这条路线能先建立“最终拿去比赛的东西长什么样”的概念。

### 想理解无人机飞控

进入 [Air 最新总文档](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/README.md)，按“软件架构 -> IMU -> 高度 -> 遥控器/CRSF -> 通信”的顺序读。Air 文档会再跳到具体源码和实验记录。

### 想理解图像到路径规划

进入 Air 的[相机模型标定](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/docs/04-competition/camera-model-calibration.md)，再看 [CarPlan3 上位机调试流程](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/docs/04-competition/car-plan3-debug-workflow.md)。这两篇会解释为什么不能只在像素域里看一条曲线，以及如何通过日志回放区分视觉、几何、规划和底盘问题。

### 想复现上位机调试

先看[上位机调试视频](https://www.bilibili.com/video/BV1Rm4m6fEMv/)，再进入 [BeaconImageAnalyzer 最新主文档](https://github.com/ZhangStudyLife/BeaconImageAnalyzer/blob/main/README.md)。如果要看 CarPlan3 专用版本，进入它的 [`car_plan` 分支 README](https://github.com/ZhangStudyLife/BeaconImageAnalyzer/blob/car_plan/README.md)。

## 仓库结构

```text
.
├─ hardware/                  PCB 源文件、板卡照片和硬件迭代记录
├─ structure/                 国赛、省赛结构件和结构说明
├─ CYT4BB7_Air/               空中控制端代码和飞控文档
├─ CYT2BL3_Image/             摄像头采集与图像识别代码
├─ BeaconImageAnalyzer/       WiFi/JustFloat 接收、记录和回放工具
├─ CYT4bb7_Car/               车端代码
└─ CYT4BB7_Car_F/             另一套车端国赛工程
```

`hardware/` 和 `structure/` 是母仓库自己的内容；Air、Image、Car 和上位机是独立子仓库。不要在 Air 里寻找 PCB 源文件，也不要把总仓库固定的子模块提交当成远端子仓库的最新版本。

## 克隆和版本关系

首次下载当前完整工程：

```bash
git clone --branch national-2026 --depth 1 --recurse-submodules --shallow-submodules https://github.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield.git
```

已经克隆后，同步总仓库记录的固定子模块版本：

```bash
git pull --recurse-submodules
git submodule update --init --recursive
```

如果只是想看子仓库远端的最新内容，请使用上面“核心入口”里的 GitHub 最新入口，不要直接在本地执行 `git submodule update --remote` 后就认为它和总仓库已经匹配。主动更新子仓库后，必须检查接口兼容性，再决定是否提交新的子模块指针。

## 文档阅读约定

- 每个重要目录都有一个总文档，负责列清单、给出视频和阅读路线。
- 具体 Markdown 负责讲实现、实验过程和个人取舍，不在总 README 里重复粘贴代码。
- 正式文档末尾应能返回所在子仓库总文档和本总仓库。
- 图片放在所属主题目录，使用能看懂含义的文件名，不使用无法判断内容的纯时间戳作为唯一名字。
- 新增视频或文档时，先更新对应总文档入口，再补充正文，避免资料已经存在但读者找不到。

## 开源状态

- 比赛年份、队伍信息、获奖证明和许可证：[待填写]
- 当前哪些代码是最终比赛版本，哪些是实验或历史版本：[待填写]
- 第三方库、参考项目和 AI 辅助说明：[待填写]

[Air 飞控最新总文档](https://github.com/ZhangStudyLife/CYT4BB7_Air/blob/national-2026/README.md) · [硬件 PCB 总文档](./hardware/README.md) · [结构总文档](./structure/README.md)
