# EFR32BG24 Sniffer 技术实现指南

## 目录

1. [硬件配置](#硬件配置)
2. [软件架构](#软件架构)
3. [连接参数解析](#连接参数解析)
4. [多连接调度算法](#多连接调度算法)
5. [数据包解析](#数据包解析)
6. [性能优化](#性能优化)

## 硬件配置

### EFR32BG24A010F1024IM40 规格

| 参数 | 规格 |
|------|------|
| CPU | ARM Cortex-M33 @ 39 MHz |
| Flash | 1024 KB |
| RAM | 192 KB |
| 射频频率 | 2.4 GHz |
| 发射功率 | +10 dBm |
| 接收灵敏度 | -98 dBm @ 125 kbps |
| 支持协议 | Bluetooth 5.3, Bluetooth Mesh |

### 引脚配置

```c
// GPIO配置示例
#define SNIFFER_LED_PORT    gpioPortA
#define SNIFFER_LED_PIN     0
#define SNIFFER_BUTTON_PORT gpioPortB
#define SNIFFER_BUTTON_PIN  0

void init_gpio(void) {
    CMU_ClockEnable(cmuClock_GPIO, true);
    
    // LED指示灯
    GPIO_PinModeSet(SNIFFER_LED_PORT, SNIFFER_LED_PIN, 
                    gpioModePushPull, 0);
    
    // 按钮输入
    GPIO_PinModeSet(SNIFFER_BUTTON_PORT, SNIFFER_BUTTON_PIN, 
                    gpioModeInputPull, 1);
}
```

## 软件架构

### 系统模块划分

```
┌─────────────────────────────────────────┐
│         应用层 (Application)            │
│  - 用户界面                              │
│  - 数据导出                              │
│  - 统计分析                              │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│      协议栈层 (Protocol Stack)          │
│  - 数据包解析                            │
│  - 连接管理                              │
│  - 加密/解密                             │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│        RAIL层 (Radio Abstraction)       │
│  - 射频控制                              │
│  - 信道切换                              │
│  - 接收过滤                              │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│          硬件层 (Hardware)              │
│  - 射频收发器                            │
│  - 定时器                                │
│  - DMA                                   │
└─────────────────────────────────────────┘
```

### 状态机设计

```c
typedef enum {
    SNIFFER_STATE_IDLE,           // 空闲状态
    SNIFFER_STATE_SCANNING,       // 扫描广播
    SNIFFER_STATE_WAIT_CONNECT,   // 等待连接请求
    SNIFFER_STATE_TRACKING,       // 跟踪连接
    SNIFFER_STATE_SYNC_LOST,      // 失去同步
    SNIFFER_STATE_ERROR           // 错误状态
} sniffer_state_t;

typedef struct {
    sniffer_state_t current_state;
    sniffer_state_t previous_state;
    uint32_t state_entry_time;
} sniffer_state_machine_t;

void sniffer_state_transition(sniffer_state_machine_t* sm, 
                              sniffer_state_t new_state) {
    sm->previous_state = sm->current_state;
    sm->current_state = new_state;
    sm->state_entry_time = get_timestamp();
    
    // 执行状态转换处理
    switch(new_state) {
        case SNIFFER_STATE_SCANNING:
            start_scanning();
            break;
        case SNIFFER_STATE_TRACKING:
            enable_connection_tracking();
            break;
        case SNIFFER_STATE_SYNC_LOST:
            attempt_resync();
            break;
        default:
            break;
    }
}
```

## 连接参数解析

### CONNECT_REQ 数据包结构

```c
typedef struct __attribute__((packed)) {
    uint8_t  pdu_type;              // PDU类型
    uint8_t  tx_add : 1;            // 发送地址类型
    uint8_t  rx_add : 1;            // 接收地址类型
    uint8_t  rfu : 6;               // 保留
    uint8_t  length;                // 长度
    uint8_t  init_addr[6];          // 发起者地址
    uint8_t  adv_addr[6];           // 广播者地址
    
    // LLData部分
    uint32_t access_address;        // 访问地址
    uint32_t crc_init : 24;         // CRC初始化
    uint8_t  win_size;              // 窗口大小
    uint16_t win_offset;            // 窗口偏移
    uint16_t interval;              // 连接间隔
    uint16_t latency;               // 从设备延迟
    uint16_t timeout;               // 超时
    uint8_t  channel_map[5];        // 信道映射
    uint8_t  hop : 5;               // 跳频增量
    uint8_t  sca : 3;               // 睡眠时钟精度
} ble_connect_req_t;

void parse_connect_req(uint8_t* packet, connection_context_t* ctx) {
    ble_connect_req_t* req = (ble_connect_req_t*)packet;
    
    // 提取连接参数
    ctx->access_address = req->access_address;
    ctx->crc_init = req->crc_init;
    ctx->hop_increment = req->hop;
    ctx->interval = req->interval;
    
    // 解析信道映射
    for(int i = 0; i < 37; i++) {
        ctx->channel_map[i] = (req->channel_map[i/8] >> (i%8)) & 0x01;
    }
    
    // 计算第一个数据信道
    ctx->current_channel = 0;
    ctx->event_counter = 0;
    
    // 计算第一个连接事件的时间
    ctx->next_event_timestamp = get_timestamp() + 
                                (req->win_offset + req->win_size) * 1250;
}
```

### 信道映射处理

```c
typedef struct {
    uint8_t num_used_channels;      // 使用的信道数
    uint8_t used_channels[37];      // 使用的信道列表
    uint16_t channel_map_bitmap;    // 位图表示
} channel_map_info_t;

void build_channel_map(uint8_t channel_map[37], 
                      channel_map_info_t* info) {
    info->num_used_channels = 0;
    info->channel_map_bitmap = 0;
    
    for(int i = 0; i < 37; i++) {
        if(channel_map[i]) {
            info->used_channels[info->num_used_channels++] = i;
            info->channel_map_bitmap |= (1 << i);
        }
    }
}

uint8_t get_data_channel(uint16_t event_counter,
                        uint8_t hop_increment,
                        channel_map_info_t* map_info) {
    // 蓝牙LE信道选择算法#1
    uint8_t unmapped = (event_counter * hop_increment) % 37;
    
    // 检查未映射信道是否在信道映射中
    if(unmapped < 37 && (map_info->channel_map_bitmap & (1 << unmapped))) {
        return unmapped;  // 信道可用，直接返回
    }
    
    // 信道不可用，需要重新映射
    // 根据蓝牙规范，使用已用信道数进行模运算
    if(map_info->num_used_channels > 0) {
        uint8_t remapping_index = unmapped % map_info->num_used_channels;
        return map_info->used_channels[remapping_index];
    }
    
    return 0;  // 默认返回信道0（不应该发生）
}
```

## 多连接调度算法

### 优先级调度

```c
typedef enum {
    PRIORITY_HIGH = 0,
    PRIORITY_MEDIUM = 1,
    PRIORITY_LOW = 2
} connection_priority_t;

typedef struct {
    connection_context_t* connection;
    uint64_t next_event_time;
    connection_priority_t priority;
} scheduled_connection_t;

// 优先级队列
typedef struct {
    scheduled_connection_t items[MAX_CONNECTIONS];
    uint8_t count;
} connection_scheduler_t;

void schedule_next_connection(connection_scheduler_t* scheduler) {
    if(scheduler->count == 0) return;
    
    // 找到下一个最早的事件
    uint8_t next_index = 0;
    uint64_t earliest_time = scheduler->items[0].next_event_time;
    
    for(uint8_t i = 1; i < scheduler->count; i++) {
        if(scheduler->items[i].next_event_time < earliest_time) {
            earliest_time = scheduler->items[i].next_event_time;
            next_index = i;
        }
    }
    
    // 如果有相同时间的事件，按优先级选择
    for(uint8_t i = 0; i < scheduler->count; i++) {
        if(scheduler->items[i].next_event_time == earliest_time &&
           scheduler->items[i].priority < scheduler->items[next_index].priority) {
            next_index = i;
        }
    }
    
    // 配置射频以监听该连接
    configure_for_connection(scheduler->items[next_index].connection);
}
```

### 时间窗口预测

```c
typedef struct {
    connection_context_t* connection;
    uint64_t start_time;
    uint64_t end_time;
    uint32_t connection_event_length;  // 连接事件持续时间(us)
} time_window_t;

bool windows_overlap(time_window_t* w1, time_window_t* w2) {
    return !(w1->end_time < w2->start_time || 
             w2->end_time < w1->start_time);
}

void detect_conflicts(connection_scheduler_t* scheduler) {
    time_window_t windows[MAX_CONNECTIONS];
    
    // 为每个连接计算时间窗口
    for(uint8_t i = 0; i < scheduler->count; i++) {
        connection_context_t* ctx = scheduler->items[i].connection;
        windows[i].start_time = ctx->next_event_timestamp;
        windows[i].connection_event_length = 2500;  // 典型值2.5ms
        windows[i].end_time = ctx->next_event_timestamp + 
                             windows[i].connection_event_length;
        windows[i].connection = ctx;
    }
    }
    
    // 检测冲突
    for(uint8_t i = 0; i < scheduler->count; i++) {
        for(uint8_t j = i + 1; j < scheduler->count; j++) {
            if(windows_overlap(&windows[i], &windows[j])) {
                handle_window_conflict(&windows[i], &windows[j]);
            }
        }
    }
}
```

## 数据包解析

### LL数据包格式

**注意**: 以下示例说明数据包结构，控制PDU处理函数需要实现。

```c
typedef struct __attribute__((packed)) {
    // 前导码 (由硬件处理)
    // 访问地址 (由硬件处理)
    
    // PDU头部
    uint8_t llid : 2;        // LL标识符
    uint8_t nesn : 1;        // 下一个期望序列号
    uint8_t sn : 1;          // 序列号
    uint8_t md : 1;          // 更多数据
    uint8_t rfu : 3;         // 保留
    uint8_t length;          // 有效载荷长度
    
    // 有效载荷
    uint8_t payload[255];
    
    // CRC (由硬件处理)
} ble_ll_data_pdu_t;

void parse_ll_data_packet(uint8_t* packet, 
                         connection_context_t* ctx) {
    ble_ll_data_pdu_t* pdu = (ble_ll_data_pdu_t*)packet;
    
    // 检查LLID
    switch(pdu->llid) {
        case 0x01:  // LL数据PDU, 继续或结束
            // process_data_pdu(pdu->payload, pdu->length);
            break;
            
        case 0x02:  // LL数据PDU, 开始
            // process_data_start(pdu->payload, pdu->length);
            break;
            
        case 0x03:  // LL控制PDU
            // process_control_pdu(pdu->payload, pdu->length);
            break;
            
        default:
            // 保留
            break;
    }
    
    // 更新序列号追踪
    ctx->last_sn = pdu->sn;
    ctx->last_nesn = pdu->nesn;
}
```

### L2CAP数据包解析

**注意**: 以下函数调用为占位符，需要根据实际需求实现。

```c
typedef struct __attribute__((packed)) {
    uint16_t length;         // 有效载荷长度
    uint16_t channel_id;     // 信道ID
    uint8_t  payload[];      // 有效载荷
} l2cap_header_t;

void parse_l2cap_packet(uint8_t* data, uint16_t length) {
    l2cap_header_t* l2cap = (l2cap_header_t*)data;
    
    printf("L2CAP Channel ID: 0x%04X, Length: %d\n", 
           l2cap->channel_id, l2cap->length);
    
    // 根据信道ID处理 (需要实现各处理函数)
    switch(l2cap->channel_id) {
        case 0x0004:  // ATT
            // parse_att_packet(l2cap->payload, l2cap->length);
            break;
            
        case 0x0005:  // L2CAP信令
            // parse_l2cap_signaling(l2cap->payload, l2cap->length);
            break;
            
        case 0x0006:  // SMP
            // parse_smp_packet(l2cap->payload, l2cap->length);
            break;
            
        default:
            if(l2cap->channel_id >= 0x0040) {
                // 动态信道
                // parse_dynamic_channel(l2cap->channel_id, 
                //                     l2cap->payload, 
                //                     l2cap->length);
            }
            break;
    }
}
```

## 性能优化

### DMA使用

```c
// 使用DMA提高数据传输效率
void setup_dma_for_packet_capture(void) {
    LDMA_Init_t init = LDMA_INIT_DEFAULT;
    LDMA_Init(&init);
    
    LDMA_TransferCfg_t transfer = 
        LDMA_TRANSFER_CFG_PERIPHERAL(ldmaPeripheralSignal_USART0_RXDATAV);
    
    LDMA_Descriptor_t descriptor = 
        LDMA_DESCRIPTOR_LINKREL_P2M_BYTE(
            &(USART0->RXDATA),
            packet_buffer,
            PACKET_BUFFER_SIZE,
            0
        );
    
    LDMA_StartTransfer(DMA_CHANNEL, &transfer, &descriptor);
}
```

### 缓冲区管理

```c
#define PACKET_BUFFER_COUNT 16
#define PACKET_MAX_SIZE 256

typedef struct {
    uint8_t data[PACKET_MAX_SIZE];
    uint16_t length;
    uint32_t access_address;
    uint64_t timestamp;
    uint8_t channel;
    int8_t rssi;
    bool in_use;
} packet_buffer_t;

typedef struct {
    packet_buffer_t buffers[PACKET_BUFFER_COUNT];
    uint8_t write_index;
    uint8_t read_index;
    uint8_t count;
} ring_buffer_t;

packet_buffer_t* allocate_packet_buffer(ring_buffer_t* rb) {
    if(rb->count >= PACKET_BUFFER_COUNT) {
        return NULL;  // 缓冲区满
    }
    
    packet_buffer_t* buffer = &rb->buffers[rb->write_index];
    rb->write_index = (rb->write_index + 1) % PACKET_BUFFER_COUNT;
    rb->count++;
    buffer->in_use = true;
    
    return buffer;
}

void free_packet_buffer(ring_buffer_t* rb) {
    if(rb->count == 0) return;
    
    rb->buffers[rb->read_index].in_use = false;
    rb->read_index = (rb->read_index + 1) % PACKET_BUFFER_COUNT;
    rb->count--;
}
```

### 功耗优化

```c
// 在连接事件之间进入低功耗模式
void enter_low_power_between_events(void) {
    // 计算到下一个事件的时间
    uint64_t current_time = get_timestamp();
    uint64_t next_event = calculate_next_event_time();
    uint32_t sleep_duration = next_event - current_time;
    
    if(sleep_duration > MINIMUM_SLEEP_TIME) {
        // 配置唤醒定时器
        setup_wakeup_timer(sleep_duration - WAKEUP_MARGIN);
        
        // 进入EM2模式
        EMU_EnterEM2(true);
        
        // 唤醒后继续
    }
}
```

## 调试技巧

### 实时日志

```c
#define LOG_LEVEL_DEBUG 0
#define LOG_LEVEL_INFO  1
#define LOG_LEVEL_WARN  2
#define LOG_LEVEL_ERROR 3

#define CURRENT_LOG_LEVEL LOG_LEVEL_INFO

#define LOG(level, fmt, ...) \
    do { \
        if(level >= CURRENT_LOG_LEVEL) { \
            printf("[%s] " fmt "\n", log_level_str[level], ##__VA_ARGS__); \
        } \
    } while(0)

void debug_print_connection_state(connection_context_t* ctx) {
    LOG(LOG_LEVEL_DEBUG, "Connection [%08X]:", ctx->access_address);
    LOG(LOG_LEVEL_DEBUG, "  Channel: %d", ctx->current_channel);
    LOG(LOG_LEVEL_DEBUG, "  Event Counter: %u", ctx->event_counter);
    LOG(LOG_LEVEL_DEBUG, "  Next Event: %llu us", ctx->next_event_timestamp);
    LOG(LOG_LEVEL_DEBUG, "  Packets: %u", ctx->packets_captured);
}
```

### 性能监控

```c
typedef struct {
    uint32_t total_packets;
    uint32_t packets_per_connection[MAX_CONNECTIONS];
    uint32_t crc_errors;
    uint32_t missed_events;
    uint32_t buffer_overflows;
    uint64_t start_time;
} performance_metrics_t;

void print_performance_report(performance_metrics_t* metrics) {
    uint64_t runtime = get_timestamp() - metrics->start_time;
    
    printf("\n=== Performance Report ===\n");
    printf("Runtime: %llu ms\n", runtime / 1000);
    printf("Total Packets: %u\n", metrics->total_packets);
    
    if(runtime > 0) {
        printf("Packet Rate: %.2f pkt/s\n", 
               (float)metrics->total_packets / (runtime / 1000000.0));
    }
    
    if(metrics->total_packets > 0) {
        printf("CRC Errors: %u (%.2f%%)\n", 
               metrics->crc_errors,
               100.0 * metrics->crc_errors / metrics->total_packets);
    } else {
        printf("CRC Errors: %u\n", metrics->crc_errors);
    }
    
    printf("Missed Events: %u\n", metrics->missed_events);
    printf("Buffer Overflows: %u\n", metrics->buffer_overflows);
}
```

## 总结

本技术指南提供了EFR32BG24芯片Sniffer功能的深入实现细节。通过合理的软件架构、高效的调度算法和优化的数据处理，可以实现稳定可靠的多连接监听功能。

关键要点：
1. 准确解析CONNECT_REQ获取连接参数
2. 实现高效的多连接调度算法
3. 使用DMA和环形缓冲区优化性能
4. 在事件间隙利用低功耗模式节省能耗
5. 完善的日志和监控机制辅助调试

---

**版本**: 1.0  
**日期**: 2026-01-29
