# 硬件设计（Hardware）

> 车机协同系留四旋翼 · 全部自研硬件 · 第 21 届全国大学生智能汽车竞赛「飞跃雷区」组

## 板卡索引

| 板卡              | 国赛版源文件                                                                  | 迭代记录                         | 核心芯片 |
| ----------------- | ----------------------------------------------------------------------------- | -------------------------------- | -------- |
| 飞控板            | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4BB7_Air/source/CYT4BB7_%E9%A3%9E%E6%8E%A7%E6%9D%BF.epro2)         | [迭代记录](docs/CYT4BB7_Air.md)   |          |
| 车控板            | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4bb7_Car/source/CYT4BB7_%E8%BD%A6%E6%8E%A7%E6%9D%BF.epro2)         | [迭代记录](docs/CYT4bb7_Car.md)   |          |
| 双 CYT2BL3 图像板 | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT2BL3_Image/source/CYT2BL3_%E5%9B%BE%E5%83%8F%E6%9D%BF.epro2)       | [迭代记录](docs/CYT2BL3_Image.md) |          |
| 无刷电调          | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/BrushlessESC/source/AI8051U_%E6%97%A0%E5%88%B7%E7%94%B5%E8%B0%83.epro2)    | [迭代记录](docs/BrushlessESC.md)  |          |
| 双路有刷驱动      | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/BrushedDriver/source/DRV8701E_%E6%9C%89%E5%88%B7%E9%A9%B1%E5%8A%A8.epro2) | [迭代记录](docs/BrushedDriver.md) |          |
| 灯板              | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/LightBoard/source/TPS54202_%E7%81%AF%E6%9D%BF.epro2)            | [迭代记录](docs/LightBoard.md)    |          |
| FPC 陀螺仪        | [直接下载 PCB 源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/FPC-Gyro/source/ICM42688P_%E9%99%80%E8%9E%BA%E4%BB%AA.epro2)        | [迭代记录](docs/FPC-Gyro.md)      |          |

## 当前说明

- 每块板一篇迭代记录（`docs/<板卡>.md`），写该板的迭代历史与踩坑。
- 国赛版源工程（`.epro2`，立创 EDA 专业版格式）见上表，命名统一为「主要芯片名_板子名」；各板 PCB 图与实物图见 `boards/<板卡>/images/`。
