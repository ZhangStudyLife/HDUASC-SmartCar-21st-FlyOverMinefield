# 硬件设计（Hardware）

> 车机协同系留四旋翼 · 全部自研硬件 · 第 21 届全国大学生智能汽车竞赛「飞跃雷区」组

> 文档状态：目录框架已初始化，技术正文将逐步补充。

## 板卡索引

| 板卡            | 源文件目录                                         | 迭代记录                          | 核心芯片 |
| ------------- | --------------------------------------------- | ----------------------------- | ---- |
| 飞控板           | [boards/CYT4BB7_Air](boards/CYT4BB7_Air/)     | [迭代记录](docs/CYT4BB7_Air.md)   |      |
| 车控板           | [boards/CYT4bb7_Car](boards/CYT4bb7_Car/)     | [迭代记录](docs/CYT4bb7_Car.md)   |      |
| 双 CYT2BL3 图像板 | [boards/CYT2BL3_Image](boards/CYT2BL3_Image/) | [迭代记录](docs/CYT2BL3_Image.md) |      |
| 无刷电调          | [boards/BrushlessESC](boards/BrushlessESC/)   | [迭代记录](docs/BrushlessESC.md)  |      |
| 双路有刷驱动        | [boards/BrushedDriver](boards/BrushedDriver/) | [迭代记录](docs/BrushedDriver.md) |      |
| 灯板            | [boards/LightBoard](boards/LightBoard/)       | [迭代记录](docs/LightBoard.md)    |      |
| FPC 陀螺仪       | [boards/FPC-Gyro](boards/FPC-Gyro/)           | [迭代记录](docs/FPC-Gyro.md)      |      |

## 当前说明

- 每块板一篇迭代记录（`docs/<板卡>.md`），写该板的迭代历史与踩坑。
- 各板源工程（`.epro2`）、原理图 PDF、Gerber 见 `boards/<板卡>/`。
