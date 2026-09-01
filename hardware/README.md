# 硬件设计（Hardware）

> 车机协同系留四旋翼 · 全部自研硬件 · 第 21 届全国大学生智能汽车竞赛「飞跃雷区」组

## 板卡索引

| 板卡              | 国赛版源文件                                                                  | 迭代记录                         | 核心芯片 |
| ----------------- | ----------------------------------------------------------------------------- | -------------------------------- | -------- |
| 飞控板            | [CYT4BB7_飞控板.epro2](boards/CYT4BB7_Air/source/CYT4BB7_飞控板.epro2)         | [迭代记录](docs/CYT4BB7_Air.md)   |          |
| 车控板            | [CYT4BB7_车控板.epro2](boards/CYT4bb7_Car/source/CYT4BB7_车控板.epro2)         | [迭代记录](docs/CYT4bb7_Car.md)   |          |
| 双 CYT2BL3 图像板 | [CYT2BL3_图像板.epro2](boards/CYT2BL3_Image/source/CYT2BL3_图像板.epro2)       | [迭代记录](docs/CYT2BL3_Image.md) |          |
| 无刷电调          | [AI8051U_无刷电调.epro2](boards/BrushlessESC/source/AI8051U_无刷电调.epro2)    | [迭代记录](docs/BrushlessESC.md)  |          |
| 双路有刷驱动      | [DRV8701E_有刷驱动.epro2](boards/BrushedDriver/source/DRV8701E_有刷驱动.epro2) | [迭代记录](docs/BrushedDriver.md) |          |
| 灯板              | [TPS54202_灯板.epro2](boards/LightBoard/source/TPS54202_灯板.epro2)            | [迭代记录](docs/LightBoard.md)    |          |
| FPC 陀螺仪        | [ICM42688P_陀螺仪.epro2](boards/FPC-Gyro/source/ICM42688P_陀螺仪.epro2)        | [迭代记录](docs/FPC-Gyro.md)      |          |

## 当前说明

- 每块板一篇迭代记录（`docs/<板卡>.md`），写该板的迭代历史与踩坑。
- 国赛版源工程（`.epro2`，立创 EDA 专业版格式）见上表，命名统一为「主要芯片名_板子名」；各板 PCB 图与实物图见 `boards/<板卡>/images/`。
