# EFR32BG24 Sniffer 快速入门指南

## 前提条件

### 硬件要求
- EFR32BG24 开发板 (如：BRD4186C/BRD4187C)
- Wireless Starter Kit (WSTK) 主板
- USB 线缆
- 待监听的蓝牙设备（至少2个）

### 软件要求
- Simplicity Studio 5 或更高版本
- Gecko SDK Suite
- Network Analyzer (可选，用于数据包分析)

## 快速开始

### 步骤 1: 环境设置

1. **安装 Simplicity Studio**
   ```bash
   # 从 Silicon Labs 官网下载
   # https://www.silabs.com/developers/simplicity-studio
   ```

2. **连接开发板**
   - 将 EFR32BG24 开发板插入 WSTK 主板
   - 通过 USB 连接到计算机
   - 确认设备被识别

3. **验证连接**
   - 打开 Simplicity Studio
   - 在 Debug Adapters 窗口中找到你的设备
   - 记录设备的序列号

### 步骤 2: 创建 Sniffer 项目

1. **新建项目**
   ```
   File → New → Silicon Labs Project Wizard
   ```

2. **选择目标**
   - Target Board: BRD4186C (或你的开发板型号)
   - Target Device: EFR32BG24A010F1024IM40

3. **选择示例**
   - 搜索 "Bluetooth - Sniffer" 或从头创建空白项目
   - 如果使用示例，跳到步骤 3

### 步骤 3: 基础代码实现

创建 `app.c` 文件：

**注意**: 以下代码为示例框架，需要补充完整的函数实现才能编译运行。

```c
#include "em_device.h"
#include "em_chip.h"
#include "em_cmu.h"
#include "rail.h"
#include "rail_ble.h"

// 连接上下文结构 (需要完整定义)
typedef struct {
    uint32_t access_address;
    uint8_t hop_increment;
    uint16_t interval;
    uint16_t channel_map[37];
    uint8_t current_channel;
    uint32_t event_counter;
    uint64_t next_event_timestamp;
    uint32_t packets_captured;
} connection_context_t;

// 全局变量
static RAIL_Handle_t rail_handle;
static RAIL_Config_t rail_config = RAIL_CONFIG_DEFAULT;
static connection_context_t active_connections[4];
static uint8_t connection_count = 0;

// 初始化函数
void app_init(void) {
    // 芯片初始化
    CHIP_Init();
    
    // 时钟初始化
    CMU_ClockEnable(cmuClock_GPIO, true);
    
    // RAIL 初始化
    rail_handle = RAIL_Init(&rail_config, NULL);
    
    // 配置 BLE
    RAIL_BLE_Init(rail_handle);
    
    // 配置接收选项
    RAIL_ConfigRxOptions(rail_handle,
                         RAIL_RX_OPTION_STORE_CRC |
                         RAIL_RX_OPTION_IGNORE_CRC_ERRORS);
    
    // 开始扫描广播信道
    start_advertising_scan();
    
    printf("Sniffer 已启动，等待连接...\n");
}

// 开始扫描广播信道
void start_advertising_scan(void) {
    // 注意: 实际实现需要使用定时器或扫描模式在信道间切换
    // 这里仅为示例，展示基本概念
    RAIL_Idle(rail_handle, RAIL_IDLE_ABORT, true);
    RAIL_StartRx(rail_handle, 37, NULL);  // 开始监听信道37
    printf("开始监听广播信道\n");
}

// RAIL 事件处理
void RAIL_EVENT_Handler(RAIL_Handle_t handle, RAIL_Events_t events) {
    if (events & RAIL_EVENT_RX_PACKET_RECEIVED) {
        handle_rx_packet(handle);
    }
    
    if (events & RAIL_EVENT_RX_PACKET_ABORTED) {
        printf("数据包接收中止\n");
    }
}

// 处理接收到的数据包
void handle_rx_packet(RAIL_Handle_t handle) {
    RAIL_RxPacketInfo_t packet_info;
    RAIL_RxPacketHandle_t packet_handle;
    uint8_t packet_buffer[256];
    
    // 获取数据包
    packet_handle = RAIL_GetRxPacketInfo(handle, 
                                         RAIL_RX_PACKET_HANDLE_OLDEST,
                                         &packet_info);
    
    if (packet_handle == RAIL_RX_PACKET_HANDLE_INVALID) {
        return;
    }
    
    // 读取数据包
    uint16_t length = RAIL_GetRxPacketData(handle, 
                                           packet_handle,
                                           packet_buffer);
    
    // 检查是否是 CONNECT_REQ (检查PDU类型)
    if (packet_buffer[0] == 0x05) {  // CONNECT_REQ PDU type
        handle_connect_req(packet_buffer, length);
    } else if (length >= 4) {
        // 检查是否属于已跟踪的连接
        uint32_t access_addr = (packet_buffer[3] << 24) | 
                              (packet_buffer[2] << 16) |
                              (packet_buffer[1] << 8) | 
                              packet_buffer[0];
        
        // 查找匹配的连接 (简化实现)
        for(uint8_t i = 0; i < connection_count; i++) {
            if(active_connections[i].access_address == access_addr) {
                active_connections[i].packets_captured++;
                printf("连接 [%08X]: 接收 %d 字节\n", access_addr, length);
                break;
            }
        }
    }
    
    // 释放数据包
    RAIL_ReleaseRxPacket(handle, packet_handle);
}

// 处理 CONNECT_REQ 数据包
void handle_connect_req(uint8_t* packet, uint16_t length) {
    if (connection_count >= 4) {
        printf("已达到最大连接数\n");
        return;
    }
    
    connection_context_t* ctx = &active_connections[connection_count];
    
    // 简化的参数解析 (实际需要完整解析CONNECT_REQ格式)
    // CONNECT_REQ格式: PDU头 + 发起者地址(6) + 广播者地址(6) + LLData(22)
    if(length >= 34) {
        // 提取访问地址 (从偏移12开始)
        ctx->access_address = (packet[15] << 24) | (packet[14] << 16) |
                             (packet[13] << 8) | packet[12];
        
        // 提取跳频增量和间隔 (简化)
        ctx->hop_increment = packet[33] & 0x1F;
        ctx->interval = (packet[21] << 8) | packet[20];
        
        // 打印连接信息
        printf("\n[新连接 #%d]\n", connection_count + 1);
        printf("访问地址: 0x%08X\n", ctx->access_address);
        printf("跳频增量: %d\n", ctx->hop_increment);
        printf("连接间隔: %d (%.2f ms)\n", 
               ctx->interval, ctx->interval * 1.25);
        
        connection_count++;
    }
}

// 主循环
void app_process_action(void) {
    // 简化的连接调度示例
    // 实际实现需要精确的时序管理和信道切换
    
    // 这里只是演示框架，完整实现需要:
    // 1. 定时器管理连接事件
    // 2. 信道跳频算法
    // 3. 时间同步机制
    // 4. 优先级调度
    
    // 周期性检查各连接状态
    for(uint8_t i = 0; i < connection_count; i++) {
        // 连接状态维护代码
    }
}

int main(void) {
    app_init();
    
    while(1) {
        app_process_action();
    }
}
```

### 步骤 4: 配置和编译

1. **配置项目**
   ```
   右键项目 → Properties → C/C++ Build → Settings
   - 确认优化级别: -O2
   - 添加必要的库: RAIL, emlib
   ```

2. **编译项目**
   ```
   Project → Build Project
   或按 Ctrl+B
   ```

3. **验证编译**
   - 确认无错误
   - 记录二进制文件大小

### 步骤 5: 烧录和测试

1. **烧录固件**
   ```
   右键项目 → Debug As → Silicon Labs ARM Program
   ```

2. **打开串口监视器**
   ```bash
   # 在 Simplicity Studio 中
   Tools → Device Console
   
   # 或使用外部工具
   # Windows: PuTTY, Tera Term
   # Linux/Mac: screen, minicom
   
   # 设置: 115200 8N1
   ```

3. **启动测试设备**
   - 开启第一个蓝牙设备（如智能手环）
   - 开启第二个蓝牙设备（如心率监测器）
   - 让它们建立连接

4. **观察输出**
   ```
   Sniffer 已启动，等待连接...
   监听广播信道 37
   监听广播信道 38
   监听广播信道 39
   
   [新连接 #1]
   访问地址: 0x8E89BED6
   跳频增量: 7
   连接间隔: 24 (30.00 ms)
   
   连接 [0x8E89BED6]: 接收 27 字节
   连接 [0x8E89BED6]: 接收 15 字节
   ...
   ```

## 常见问题解决

### 问题 1: 无法捕获 CONNECT_REQ

**原因**:
- 连接建立时 Sniffer 未在正确的广播信道
- 信号太弱

**解决方案**:
```c
// 增加扫描时间
void scan_all_advertising_channels(void) {
    for(int round = 0; round < 10; round++) {
        for(uint8_t ch = 37; ch <= 39; ch++) {
            RAIL_StartRx(rail_handle, ch, NULL);
            delay_ms(100);  // 在每个信道停留更长时间
        }
    }
}
```

### 问题 2: 快速失去同步

**原因**:
- 时钟漂移
- 计算错误

**解决方案**:
```c
// 添加时间容差
#define TIME_TOLERANCE_US 150

bool is_event_time(connection_context_t* ctx) {
    uint64_t current_time = get_timestamp();
    int64_t diff = ctx->next_event_timestamp - current_time;
    
    // 提前一点切换到接收状态
    return (diff < TIME_TOLERANCE_US && diff > -TIME_TOLERANCE_US);
}
```

### 问题 3: 多连接时丢包

**原因**:
- 连接事件时间冲突
- 缓冲区不足

**解决方案**:
```c
// 实现智能调度
void schedule_connections(void) {
    // 按下一个事件时间排序
    sort_connections_by_next_event();
    
    // 检测冲突
    for(int i = 0; i < connection_count - 1; i++) {
        if(events_overlap(i, i+1)) {
            // 选择优先级高的
            skip_lower_priority_event(i, i+1);
        }
    }
}
```

## 数据包分析

### 使用 Network Analyzer

1. **启用日志输出**
   ```c
   // 将数据包发送到 PTI (Packet Trace Interface)
   RAIL_ConfigPti(rail_handle, &pti_config);
   ```

2. **启动 Network Analyzer**
   ```
   Tools → Network Analyzer
   选择你的设备
   点击 Start Capture
   ```

3. **查看数据包**
   - 过滤特定访问地址
   - 分析协议层
   - 导出为 pcap 格式

### 导出数据

```c
// 通过串口输出 CSV 格式
void export_packet_csv(captured_packet_t* pkt) {
    printf("%llu,%08X,%d,%d,%d\n",
           pkt->timestamp,
           pkt->access_address,
           pkt->channel,
           pkt->rssi,
           pkt->length);
}
```

## 性能优化建议

### 1. 限制连接数
```c
#define MAX_SIMULTANEOUS_CONNECTIONS 2
// 同时跟踪不超过 2 个连接以获得最佳性能
```

### 2. 使用 DMA
```c
// 启用 DMA 减少 CPU 负载
RAIL_ConfigDma(rail_handle, &dma_config);
```

### 3. 优化缓冲区
```c
// 使用合适的缓冲区大小
#define PACKET_BUFFER_SIZE 512
#define BUFFER_COUNT 16
```

## 下一步

- 实现数据包过滤和解析
- 添加 GUI 界面
- 支持更多连接
- 实现数据包重组
- 添加统计和可视化

## 参考资源

- [EFR32BG24 数据手册](https://www.silabs.com/documents/public/data-sheets/efr32bg24-datasheet.pdf)
- [RAIL API 文档](https://docs.silabs.com/rail/latest/)
- [Bluetooth LE 规范](https://www.bluetooth.com/specifications/specs/)
- [Simplicity Studio 用户指南](https://docs.silabs.com/simplicity-studio-5-users-guide/latest/)

---

**版本**: 1.0  
**最后更新**: 2026-01-29
