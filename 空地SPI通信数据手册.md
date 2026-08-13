# 空地 SPI 通信数据手册

## 1. 协议概要

| 参数 | 值 |
|------|-----|
| 硬件 | CYT4BB7 Air SCB6 (Master) ↔ CYT2BL3 SCB0 (Slave) × 2 |
| 时钟 | 10 MHz, CPOL0_CPHA0 |
| 交换长度 | 32 字节/帧，全双工 |
| 有效载荷 | **12 字节/帧**（其余为协议开销） |
| 从机轮询 | 每 5ms 一轮（200Hz），每轮推进 2 板的非阻塞事务 |
| 读取频率 | 每块板由 Camera SPI 服务按 200Hz 调度，结果按实际完成帧驱动 |
| 发送频率 | 每块板由 Camera SPI 服务按 200Hz 调度 |

帧格式（32 字节）：

```
AA 55 0x20 [len_H len_L] [payload...] [CRC16_L CRC16_H] ED
 2B head   1B cmd  2B BE        N bytes      2B LE        1B tail
```

CRC16-MODBUS 覆盖范围：`cmd` 到 `payload` 末尾。

## 2. API 速查

### 主机端（CYT4bb7_Car）

```c
#include "Protocols/CameraSpi/camera_spi.h"

CameraSpi_Init();                                          // 初始化（car_loop_init 中已调用）
CameraSpi_Poll();                                          // 轮询驱动（car_loop_100HZ 中已调用）

// 发送：向从机 id 发送 data[0..len-1]，len ≤ 12
CameraSpi_SendRaw(CAMERA_SPI_SLAVE_0, data, 12);

// 接收：读取从机 id 的最新数据，返回 1=有数据，0=离线
uint8 rx[12];
uint16 rx_len = sizeof(rx);
if (CameraSpi_ReceiveRaw(CAMERA_SPI_SLAVE_0, rx, &rx_len)) {
    // rx[0..rx_len-1] 就是从机发来的数据
}
```

### 从机端（CYT2BL3_Image）

```c
#include "Protocols/CameraSpi/camera_spi_slave.h"

CameraSpiSlave_Init();                                     // 初始化（main 中已调用）
CameraSpiSlave_Task();                                     // 主循环轮询（可为空）

// 发送：向主机发送 data[0..len-1]，len ≤ 12
CameraSpiSlave_SendRaw(data, 12);

// 接收：读取主机下发的最新数据，返回 1=有新数据
uint8 rx[12];
uint16 rx_len = sizeof(rx);
if (CameraSpiSlave_ReceiveRaw(rx, &rx_len)) {
    // rx[0..rx_len-1] 就是主机发来的数据
}
```

## 3. 当前测试数据格式

### 主机 → 从机（12 字节）

```
byte[0]     = 0x5A          magic
byte[1]     = slave_id      0/1/2
byte[2..5]  = tx_counter    u32 LE，累计发送次数
byte[6..11] = 0x00          填充
```

### 从机 → 主机（12 字节）

```
byte[0..1]  = frame_id      u16 LE，图像帧序号
byte[2..3]  = spot_count    u16 LE，固定=1
byte[4..5]  = x             u16 LE，= 图像宽度/2（测试值）
byte[6..7]  = y             u16 LE，= 图像高度/2（测试值）
byte[8..9]  = area          u16 LE，= 宽×高（测试值）
byte[10..11]= 0x00          填充
```

## 4. 如何扩展自定义数据

12 字节是你的自由空间，两端自行约定格式。建议：

```
byte[0]     = kind          消息类型（0x01=目标, 0x02=状态, ...）
byte[1..11] = payload       自定义字段，小端序
```

### 示例：从机发送真实信标检测结果

```c
// 从机端（main_cm4.c 中调用）
void image_send_beacon_result(uint16 x, uint16 y, uint16 area)
{
    uint8 tx[12] = {0};
    tx[0] = 0x01;                           // kind = 目标
    tx[1] = (uint8)(x & 0xFF);              // x low
    tx[2] = (uint8)(x >> 8);                // x high
    tx[3] = (uint8)(y & 0xFF);              // y low
    tx[4] = (uint8)(y >> 8);                // y high
    tx[5] = (uint8)(area & 0xFF);           // area low
    tx[6] = (uint8)(area >> 8);             // area high
    CameraSpiSlave_SendRaw(tx, 12);
}
```

### 示例：主机端解析

```c
// 主机端（car_loop 中调用）
uint8 rx[12];
uint16 rx_len;
if (CameraSpi_ReceiveRaw(CAMERA_SPI_SLAVE_0, rx, &rx_len) && rx[0] == 0x01) {
    uint16 x    = (uint16)rx[1] | ((uint16)rx[2] << 8);
    uint16 y    = (uint16)rx[3] | ((uint16)rx[4] << 8);
    uint16 area = (uint16)rx[5] | ((uint16)rx[6] << 8);
    // 使用 x, y, area
}
```

## 5. 关键文件索引

| 角色 | 文件 | 说明 |
|------|------|------|
| 主机 API | `CYT4bb7_Car/project/code/Protocols/CameraSpi/camera_spi.h` | 4 个公开函数 |
| 主机调用 | `CYT4bb7_Car/project/code/Controller/car_loop.c` | car_loop_100HZ 中调用示例 |
| 从机 API | `CYT2BL3_Image/project/code/Protocols/CameraSpi/camera_spi_slave.h` | 4 个公开函数 |
| 从机调用 | `CYT2BL3_Image/project/user/main_cm4.c` | main 循环中调用示例 |
| 共享协议 | 两工程各自的 `camera_spi_protocol.h` | 帧格式常量、CRC16 |
| 共享类型 | 两工程各自的 `camera_spi_types.h` | typed payload 结构体（未使用） |
