# EFR32BG24A010F1024IM40 蓝牙Sniffer功能

## 概述

EFR32BG24A010F1024IM40是Silicon Labs推出的高性能蓝牙SoC芯片。本文档详细说明该芯片的Sniffer（嗅探器）功能，以及如何使用它监听两个连接设备之间的通信。

## Sniffer功能机制

### 1. 工作原理

Sniffer模式允许EFR32BG24芯片以被动监听者的身份捕获蓝牙LE（低功耗蓝牙）设备之间的数据包，而不参与实际的连接过程。

**核心机制：**

1. **被动监听模式**：芯片配置为只接收（RX）模式，不发送任何数据包
2. **频道跟踪**：自动跟踪连接使用的跳频序列（Channel Hopping Sequence）
3. **访问地址匹配**：通过捕获的访问地址（Access Address）识别特定连接
4. **时序同步**：同步到连接事件的时序以持续捕获数据包

### 2. 监听两个连接设备的机制

当需要监听两个已连接的蓝牙设备时，Sniffer需要：

#### 步骤1：捕获连接建立过程
```
Central Device  -->  CONNECT_REQ  -->  Peripheral Device
      |                                       |
   [Sniffer芯片监听并捕获]
```

**CONNECT_REQ数据包包含关键信息：**
- Access Address（访问地址）- 连接的唯一标识符
- CRC Init（CRC初始化值）
- WinSize, WinOffset（连接窗口参数）
- Interval（连接间隔）
- Channel Map（信道映射）
- Hop Increment（跳频增量）

#### 步骤2：同步到连接
- 使用捕获的访问地址配置接收过滤器
- 根据跳频算法计算下一个信道
- 在预期的时间窗口内监听数据包

#### 步骤3：持续跟踪
- 实时计算并跳转到下一个通信信道
- 保持与连接事件的时序同步
- 捕获所有数据和控制数据包

## 如何区分监听的是哪个设备

在同时监听多个连接时，可以通过以下方式区分不同的设备：

### 1. **访问地址（Access Address）**

每个蓝牙LE连接都有唯一的32位访问地址：

```
Connection 1: Access Address = 0x8E89BED6
Connection 2: Access Address = 0x5DA45BC3
```

**实现方法：**
```c
// 伪代码示例
typedef struct {
    uint32_t access_address;
    char* device_name;
    uint8_t connection_id;
} sniffer_connection_t;

sniffer_connection_t connections[2] = {
    {0x8E89BED6, "Device_A", 1},
    {0x5DA45BC3, "Device_B", 2}
};

// 接收数据包时根据访问地址识别
void packet_received(uint32_t access_addr, uint8_t* data) {
    for(int i = 0; i < 2; i++) {
        if(connections[i].access_address == access_addr) {
            printf("从 %s 接收到数据包\n", connections[i].device_name);
            process_packet(connections[i].connection_id, data);
            break;
        }
    }
}
```

### 2. **蓝牙设备地址（BD_ADDR）**

在连接建立阶段（广播和连接请求），可以捕获设备的蓝牙地址：

```
Central Device MAC: AA:BB:CC:DD:EE:FF
Peripheral Device MAC: 11:22:33:44:55:66
```

**地址映射表：**
```c
typedef struct {
    uint8_t bd_addr[6];
    uint32_t access_address;
    char* friendly_name;
} device_mapping_t;

device_mapping_t device_map[] = {
    {{0xAA,0xBB,0xCC,0xDD,0xEE,0xFF}, 0x8E89BED6, "智能手环"},
    {{0x11,0x22,0x33,0x44,0x55,0x66}, 0x5DA45BC3, "心率传感器"}
};
```

### 3. **连接上下文管理**

使用连接上下文结构维护每个连接的状态：

```c
typedef struct {
    // 标识信息
    uint32_t access_address;
    uint8_t central_addr[6];
    uint8_t peripheral_addr[6];
    
    // 连接参数
    uint8_t hop_increment;
    uint16_t interval;
    uint16_t channel_map[37];
    
    // 状态追踪
    uint32_t event_counter;
    uint8_t current_channel;
    uint64_t next_event_timestamp;
    
    // 统计信息
    uint32_t packets_captured;
    uint32_t crc_errors;
    uint32_t missed_events;  // 添加此字段
} connection_context_t;

// 维护多个连接上下文
connection_context_t connections[MAX_CONNECTIONS];
```

### 4. **时分复用（Time Division Multiplexing）**

由于Sniffer只有一个射频接口，监听多个连接需要在时间上复用：

```
Time Line:
|--Connection 1--|  |--Connection 2--|  |--Connection 1--|  |--Connection 2--|
    Event N          Event M           Event N+1          Event M+1
```

**调度算法：**
1. 预测每个连接的下一个事件时间
2. 优先监听即将发生的事件
3. 如果事件时间冲突，根据优先级选择
4. 记录错过的事件用于后续恢复同步

### 5. **数据包标识和分类**

在捕获的数据包中添加元数据：

```c
typedef struct {
    // 原始数据
    uint8_t raw_packet[256];
    uint16_t packet_length;
    
    // 标识信息
    uint32_t access_address;
    uint8_t connection_id;
    
    // 时间戳
    uint64_t timestamp_us;
    
    // 信道信息
    uint8_t channel;
    
    // 信号质量
    int8_t rssi;
    uint8_t crc_ok;
} captured_packet_t;
```

## 实现示例

### 使用Silicon Labs工具链

```c
// 初始化Sniffer模式
void init_sniffer(void) {
    // 配置射频为接收模式
    RAIL_ConfigRxOptions(rail_handle, 
                         RAIL_RX_OPTION_STORE_CRC | 
                         RAIL_RX_OPTION_IGNORE_CRC_ERRORS);
    
    // 设置接收过滤器
    RAIL_SetRxTransitions(rail_handle, 
                          &rx_transitions);
}

// 添加连接监听
void add_connection_to_sniffer(uint32_t access_address,
                               uint8_t hop_increment,
                               uint16_t channel_map,
                               uint16_t interval) {
    connection_context_t* ctx = allocate_connection_context();
    
    ctx->access_address = access_address;
    ctx->hop_increment = hop_increment;
    ctx->interval = interval;
    
    // 配置访问地址过滤
    configure_access_address_filter(access_address);
    
    // 计算初始信道
    ctx->current_channel = calculate_first_channel(channel_map);
    
    // 添加到监听列表
    add_to_connection_list(ctx);
}

// 处理接收到的数据包
void on_packet_received(uint8_t* packet, uint16_t length) {
    uint32_t access_addr = extract_access_address(packet);
    
    connection_context_t* ctx = find_connection(access_addr);
    if (ctx != NULL) {
        // 识别是哪个连接的数据包
        printf("连接 [%08X]: 接收 %d 字节\n", 
               access_addr, length);
        
        // 更新连接状态
        ctx->event_counter++;
        ctx->packets_captured++;
        
        // 计算下一个信道
        ctx->current_channel = calculate_next_channel(
            ctx->current_channel,
            ctx->hop_increment,
            ctx->channel_map
        );
        
        // 切换到下一个信道
        RAIL_SetRxChannel(rail_handle, ctx->current_channel);
    }
}
```

## 信道跳频算法

蓝牙LE使用自适应跳频来避免干扰：

**注意**: 以下为简化示例用于说明概念。实际实现需要正确处理channel_map的位图格式。

```c
// 计算下一个数据信道 (简化示例)
// 注意: channel_map实际是5字节位图，这里简化为布尔数组以说明概念
uint8_t calculate_next_channel(uint8_t current_channel,
                                uint8_t hop_increment,
                                uint16_t channel_map[37]) {
    uint8_t unmapped_channel = (current_channel + hop_increment) % 37;
    
    // 根据信道映射找到实际使用的信道
    uint8_t mapped_channel = 0;
    uint8_t count = 0;
    
    for (int i = 0; i < 37; i++) {
        if (channel_map[i]) {
            if (count == unmapped_channel) {
                mapped_channel = i;
                break;
            }
            count++;
        }
    }
    
    return mapped_channel;
}
```

## 最佳实践

### 1. **初始化阶段**
- 先监听广播信道（37, 38, 39）捕获CONNECT_REQ
- 验证CRC以确保数据包完整性
- 保存所有连接参数

### 2. **连接追踪**
- 维持精确的时间同步
- 实现重新同步机制处理丢包情况
- 使用环形缓冲区存储数据包避免丢失

### 3. **多连接管理**
- 限制同时监听的连接数量（建议≤4）
- 实现优先级队列管理连接
- 监控每个连接的质量指标

### 4. **调试和日志**
```c
typedef struct {
    uint32_t total_packets;
    uint32_t crc_errors;
    uint32_t missed_events;  // 需要在调度逻辑中更新此计数
    uint32_t sync_losses;
} sniffer_statistics_t;

void print_statistics(connection_context_t* ctx) {
    printf("连接 [%08X] 统计:\n", ctx->access_address);
    printf("  捕获数据包: %u\n", ctx->packets_captured);
    printf("  CRC错误: %u\n", ctx->crc_errors);
    printf("  丢失事件: %u\n", ctx->missed_events);
}

// 在检测到丢失事件时调用
void on_event_missed(connection_context_t* ctx) {
    ctx->missed_events++;
}
```

## 工具和资源

### Silicon Labs官方工具
- **Simplicity Studio**: 开发环境
- **Network Analyzer**: 蓝牙协议分析工具
- **WSTK (Wireless Starter Kit)**: 开发板

### 参考文档
- [EFR32BG24 参考手册](https://www.silabs.com/wireless/bluetooth)
- [蓝牙核心规范 v5.x](https://www.bluetooth.com/specifications/specs/)
- [Silicon Labs RAIL 库文档](https://docs.silabs.com/rail/)

## 常见问题

### Q1: 为什么无法同时监听多个连接？
**A**: 单个射频接口只能在同一时刻监听一个信道。需要通过时分复用在多个连接间切换。

### Q2: 如何处理连接事件冲突？
**A**: 实现优先级调度算法，或增加硬件（使用多个Sniffer设备）。

### Q3: 丢失同步后如何恢复？
**A**: 监听已知的信道，通过匹配访问地址重新捕获数据包，利用MIC（Message Integrity Check）验证同步。

### Q4: 加密连接能监听吗？
**A**: Sniffer可以捕获加密的数据包，但无法解密内容（除非知道密钥）。可以分析元数据和时序信息。

## 示例应用场景

1. **协议调试**: 验证蓝牙协议实现的正确性
2. **性能分析**: 测量连接间隔、延迟、吞吐量
3. **故障诊断**: 分析连接断开、重连等问题
4. **互操作性测试**: 验证不同厂商设备的兼容性

## 许可和免责声明

本文档仅供教育和研究目的。使用Sniffer功能监听蓝牙通信时，请遵守当地法律法规和隐私保护规定。

---

**作者**: sample.dotblog  
**最后更新**: 2026-01-29  
**版本**: 1.0
