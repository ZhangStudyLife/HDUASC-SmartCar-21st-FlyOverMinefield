# HDUASC SmartCar 21st FlyOverMinefield

这是空地协同总项目仓库，根目录负责组织三个固件子项目，并保留总说明和联调文档。

## 项目结构

- `CYT4BB7_Air/`：飞机主控代码
- `CYT4bb7_Car/`：车主控代码
- `CYT2BL3_Image/`：车上处理飞机摄像头数据的小芯片代码
- `空地通信互传数据手册.md`：当前空地通信协议和收发实现说明

## 三个子项目职责

### 1. `CYT4BB7_Air`

飞机主控工程，负责飞机侧传感器、控制逻辑、以及和车主控之间的空地通信。

### 2. `CYT4bb7_Car`

车主控工程，负责车辆控制、状态估计、和飞机主控通信，以及和车载图像处理小芯片交换摄像头结果。

### 3. `CYT2BL3_Image`

车载图像处理小芯片工程，运行在 `CYT2BL3` 芯片上，专门处理来自飞机摄像头相关的数据，并向车主控输出处理结果。

## 仓库初始化

首次克隆总仓库时，必须连同三个子模块一起拉取：

```bash
git clone --recursive <repo-url>
```

如果已经克隆过总仓库，再初始化或补齐子模块：

```bash
git submodule update --init --recursive
```

如果需要拉取各子模块远端最新提交：

```bash
git submodule update --init --recursive --remote
```

## 版本管理约束

- 总仓库通过 `git submodule` 固定三个子项目的提交版本。
- 联调时以总仓库记录的子模块提交为准，避免三边代码版本漂移。
- 更新任意一个子项目后，需要回到总仓库提交新的 submodule 指针，才能让其他人拿到同一套联调版本。

## 日常使用建议

### 查看当前子模块状态

```bash
git submodule status
```

### 同步 `.gitmodules` 配置变更

```bash
git submodule sync --recursive
```

### 拉取总仓库后更新到记录版本

```bash
git submodule update --init --recursive
```

## 联调入口

- 空地通信协议说明：[`空地通信互传数据手册.md`](./空地通信互传数据手册.md)
- 车端摄像头链路迁移说明：[`CYT4bb7_Car/docs/air_camera_comm_migration.md`](./CYT4bb7_Car/docs/air_camera_comm_migration.md)

## 开发约定

- 根仓库负责维护总项目结构、子模块版本和联调文档。
- 三个固件工程原则上各自独立开发、独立编译，避免把跨工程改动直接揉进总仓根目录。
- 新同学接手时，先确认子模块完整拉取，再分别打开各自 IAR 工程进行编译和下载。
