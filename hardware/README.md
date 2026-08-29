# 硬件设计（Hardware）

> 车机协同系留四旋翼 · 全部自研硬件 · 第 21 届全国大学生智能汽车竞赛「飞跃雷区」组

> 文档状态：持续迭代中。下表中的迭代记录可直接在线阅读，PCB 源文件可直接下载。

## 板卡索引

| 板卡 | PCB 源文件 | 迭代记录 | 核心芯片 |
| --- | --- | --- | --- |
| 飞控板 | [下载PCB源文件（国赛迭代）](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4BB7_Air/source/ProPrj_%E9%A3%9E%E6%8E%A7%E5%9B%BD%E8%B5%9B%E8%BF%AD%E4%BB%A3_2026-08-15.epro2)<br>[下载PCB源文件（核心板 ver1.1）](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4BB7_Air/source/%E6%A0%B8%E5%BF%83%E6%9D%BF%E9%A3%9E%E6%8E%A7ver1.1.epro2)<br>[下载PCB源文件（飞控板 ver1.0）](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4BB7_Air/source/%E9%A3%9E%E6%8E%A7%E6%9D%BFver1.0.epro2)<br>[下载PCB源文件（飞控裸片 ver2.0）](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4BB7_Air/source/%E9%A3%9E%E6%8E%A7%E8%A3%B8%E7%89%87ver2.0.epro2) | [查看迭代记录](docs/CYT4bb7_Air.md) | CYT4BB7 |
| 车控板 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT4bb7_Car/source/ProPrj_%E8%BD%A6%E6%9D%BF%E8%BF%AD%E4%BB%A3_2026-08-15.epro2) | [查看迭代记录](docs/CYT4bb7_Car.md) | CYT4BB7 |
| 双 CYT2BL3 图像板 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/CYT2BL3_Image/source/ProPrj_%E5%9B%BE%E5%83%8F%E6%9D%BF%E5%9B%BD%E8%B5%9B%E8%BF%AD%E4%BB%A3_2026-08-15.epro2) | [查看迭代记录](docs/CYT2BL3_Image.md) | CYT2BL3 |
| 无刷电调 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/BrushlessESC/source/ProPrj_ai8051%E7%94%B5%E8%B0%83_2026-08-15.epro2) | [查看迭代记录](docs/BrushlessESC.md) | AI8051 |
| 双路有刷驱动 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/BrushedDriver/source/DRV8701_%E6%9C%89%E5%88%B7%E9%A9%B1%E5%8A%A8.epro2) | [查看迭代记录](docs/BrushedDriver.md) | DRV8701 |
| 灯板 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/LightBoard/source/%E9%93%9D%E5%9F%BA%E6%9D%BF%E7%BA%A2%E5%A4%96%E7%81%AF%E6%9D%BF.epro2) | [查看迭代记录](docs/LightBoard.md) | — |
| FPC 陀螺仪 | [下载PCB源文件](https://raw.githubusercontent.com/ZhangStudyLife/HDUASC-SmartCar-21st-FlyOverMinefield/national-2026/hardware/boards/FPC-Gyro/source/ICM42688P_FPC.epro2) | [查看迭代记录](docs/FPC-Gyro.md) | ICM42688P |

## 当前说明

- 迭代记录位于 [`docs/`](docs/) 目录，可在 GitHub 页面直接打开阅读。
- 表格中的“下载PCB源文件”链接指向仓库 `national-2026` 分支中的 `.epro2` 文件，点击即可下载。
- 当前仓库已提交的 PCB 源文件均为 `.epro2` 工程文件；暂未提供原理图 PDF 或 Gerber 文件。
- 已补充双路有刷驱动（DRV8701）、铝基板红外灯板和 ICM42688P FPC 陀螺仪的 PCB 源文件，分别归档在 `boards/BrushedDriver/source/`、`boards/LightBoard/source/` 和 `boards/FPC-Gyro/source/`，点击表格中的“下载PCB源文件”即可下载。
- 飞控板迭代记录所需图片位于 [`boards/CYT4BB7_Air/images/`](boards/CYT4BB7_Air/images/)。
