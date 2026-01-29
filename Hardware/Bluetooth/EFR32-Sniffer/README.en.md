# EFR32BG24A010F1024IM40 Bluetooth Sniffer Function

## Overview

The EFR32BG24A010F1024IM40 is a high-performance Bluetooth SoC chip from Silicon Labs. This document details the Sniffer functionality of this chip and explains how to use it to monitor communication between two connected devices.

## Sniffer Mechanism

### 1. Working Principle

Sniffer mode allows the EFR32BG24 chip to capture Bluetooth LE (Low Energy) packets between devices as a passive listener, without participating in the actual connection process.

**Core Mechanism:**

1. **Passive Listening Mode**: The chip is configured in receive-only (RX) mode, not transmitting any packets
2. **Channel Tracking**: Automatically tracks the channel hopping sequence used by the connection
3. **Access Address Matching**: Identifies specific connections through captured Access Addresses
4. **Timing Synchronization**: Synchronizes with connection event timing to continuously capture packets

### 2. Mechanism for Monitoring Two Connected Devices

When monitoring two connected Bluetooth devices, the Sniffer needs to:

#### Step 1: Capture Connection Establishment
```
Central Device  -->  CONNECT_REQ  -->  Peripheral Device
      |                                       |
   [Sniffer Chip Listens and Captures]
```

**CONNECT_REQ packet contains key information:**
- Access Address - Unique identifier for the connection
- CRC Init - CRC initialization value
- WinSize, WinOffset - Connection window parameters
- Interval - Connection interval
- Channel Map - Channel mapping
- Hop Increment - Frequency hopping increment

#### Step 2: Synchronize to Connection
- Configure receive filter using captured Access Address
- Calculate next channel based on hopping algorithm
- Listen for packets in expected time window

#### Step 3: Continuous Tracking
- Calculate and jump to next communication channel in real-time
- Maintain timing synchronization with connection events
- Capture all data and control packets

## How to Distinguish Which Device is Being Monitored

When monitoring multiple connections simultaneously, you can distinguish different devices through:

### 1. **Access Address**

Each Bluetooth LE connection has a unique 32-bit Access Address:

```
Connection 1: Access Address = 0x8E89BED6
Connection 2: Access Address = 0x5DA45BC3
```

**Implementation Example:**
```c
// Pseudo code example
typedef struct {
    uint32_t access_address;
    char* device_name;
    uint8_t connection_id;
} sniffer_connection_t;

sniffer_connection_t connections[2] = {
    {0x8E89BED6, "Device_A", 1},
    {0x5DA45BC3, "Device_B", 2}
};

// Identify device based on Access Address when receiving packets
void packet_received(uint32_t access_addr, uint8_t* data) {
    for(int i = 0; i < 2; i++) {
        if(connections[i].access_address == access_addr) {
            printf("Packet received from %s\n", connections[i].device_name);
            process_packet(connections[i].connection_id, data);
            break;
        }
    }
}
```

### 2. **Bluetooth Device Address (BD_ADDR)**

During connection establishment (advertising and connection request), capture device Bluetooth addresses:

```
Central Device MAC: AA:BB:CC:DD:EE:FF
Peripheral Device MAC: 11:22:33:44:55:66
```

**Address Mapping Table:**
```c
typedef struct {
    uint8_t bd_addr[6];
    uint32_t access_address;
    char* friendly_name;
} device_mapping_t;

device_mapping_t device_map[] = {
    {{0xAA,0xBB,0xCC,0xDD,0xEE,0xFF}, 0x8E89BED6, "Smart Bracelet"},
    {{0x11,0x22,0x33,0x44,0x55,0x66}, 0x5DA45BC3, "Heart Rate Monitor"}
};
```

### 3. **Connection Context Management**

Maintain connection state using connection context structures:

```c
typedef struct {
    // Identity information
    uint32_t access_address;
    uint8_t central_addr[6];
    uint8_t peripheral_addr[6];
    
    // Connection parameters
    uint8_t hop_increment;
    uint16_t interval;
    uint16_t channel_map[37];
    
    // State tracking
    uint32_t event_counter;
    uint8_t current_channel;
    uint64_t next_event_timestamp;
    
    // Statistics
    uint32_t packets_captured;
    uint32_t crc_errors;
} connection_context_t;

// Maintain multiple connection contexts
connection_context_t connections[MAX_CONNECTIONS];
```

### 4. **Time Division Multiplexing**

Since the Sniffer has only one RF interface, monitoring multiple connections requires time multiplexing:

```
Time Line:
|--Connection 1--|  |--Connection 2--|  |--Connection 1--|  |--Connection 2--|
    Event N          Event M           Event N+1          Event M+1
```

**Scheduling Algorithm:**
1. Predict next event time for each connection
2. Prioritize monitoring imminent events
3. If event times conflict, choose based on priority
4. Record missed events for later synchronization recovery

### 5. **Packet Identification and Classification**

Add metadata to captured packets:

```c
typedef struct {
    // Raw data
    uint8_t raw_packet[256];
    uint16_t packet_length;
    
    // Identification
    uint32_t access_address;
    uint8_t connection_id;
    
    // Timestamp
    uint64_t timestamp_us;
    
    // Channel information
    uint8_t channel;
    
    // Signal quality
    int8_t rssi;
    uint8_t crc_ok;
} captured_packet_t;
```

## Implementation Example

### Using Silicon Labs Toolchain

```c
// Initialize Sniffer mode
void init_sniffer(void) {
    // Configure RF for receive mode
    RAIL_ConfigRxOptions(rail_handle, 
                         RAIL_RX_OPTION_STORE_CRC | 
                         RAIL_RX_OPTION_IGNORE_CRC_ERRORS);
    
    // Set receive filters
    RAIL_SetRxTransitions(rail_handle, 
                          &rx_transitions);
}

// Add connection to sniffer
void add_connection_to_sniffer(uint32_t access_address,
                               uint8_t hop_increment,
                               uint16_t channel_map,
                               uint16_t interval) {
    connection_context_t* ctx = allocate_connection_context();
    
    ctx->access_address = access_address;
    ctx->hop_increment = hop_increment;
    ctx->interval = interval;
    
    // Configure Access Address filter
    configure_access_address_filter(access_address);
    
    // Calculate initial channel
    ctx->current_channel = calculate_first_channel(channel_map);
    
    // Add to monitoring list
    add_to_connection_list(ctx);
}

// Process received packets
void on_packet_received(uint8_t* packet, uint16_t length) {
    uint32_t access_addr = extract_access_address(packet);
    
    connection_context_t* ctx = find_connection(access_addr);
    if (ctx != NULL) {
        // Identify which connection's packet
        printf("Connection [%08X]: Received %d bytes\n", 
               access_addr, length);
        
        // Update connection state
        ctx->event_counter++;
        ctx->packets_captured++;
        
        // Calculate next channel
        ctx->current_channel = calculate_next_channel(
            ctx->current_channel,
            ctx->hop_increment,
            ctx->channel_map
        );
        
        // Switch to next channel
        RAIL_SetRxChannel(rail_handle, ctx->current_channel);
    }
}
```

## Channel Hopping Algorithm

Bluetooth LE uses adaptive frequency hopping to avoid interference:

```c
// Calculate next data channel
uint8_t calculate_next_channel(uint8_t current_channel,
                                uint8_t hop_increment,
                                uint16_t channel_map[37]) {
    uint8_t unmapped_channel = (current_channel + hop_increment) % 37;
    
    // Find actual used channel based on channel map
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

## Best Practices

### 1. **Initialization Phase**
- Monitor advertising channels (37, 38, 39) first to capture CONNECT_REQ
- Verify CRC to ensure packet integrity
- Save all connection parameters

### 2. **Connection Tracking**
- Maintain precise time synchronization
- Implement re-synchronization mechanism for packet loss
- Use circular buffer to store packets and avoid loss

### 3. **Multiple Connection Management**
- Limit number of simultaneous connections (recommend ≤4)
- Implement priority queue for connection management
- Monitor quality metrics for each connection

### 4. **Debugging and Logging**
```c
typedef struct {
    uint32_t total_packets;
    uint32_t crc_errors;
    uint32_t missed_events;  // Update this counter in scheduling logic
    uint32_t sync_losses;
} sniffer_statistics_t;

void print_statistics(connection_context_t* ctx) {
    printf("Connection [%08X] Statistics:\n", ctx->access_address);
    printf("  Captured Packets: %u\n", ctx->packets_captured);
    printf("  CRC Errors: %u\n", ctx->crc_errors);
    printf("  Missed Events: %u\n", ctx->missed_events);
}

// Call when an event is missed
void on_event_missed(connection_context_t* ctx) {
    ctx->missed_events++;
}
```

## Tools and Resources

### Silicon Labs Official Tools
- **Simplicity Studio**: Development environment
- **Network Analyzer**: Bluetooth protocol analysis tool
- **WSTK (Wireless Starter Kit)**: Development board

### Reference Documentation
- [EFR32BG24 Reference Manual](https://www.silabs.com/wireless/bluetooth)
- [Bluetooth Core Specification v5.x](https://www.bluetooth.com/specifications/specs/)
- [Silicon Labs RAIL Library Documentation](https://docs.silabs.com/rail/)

## FAQ

### Q1: Why can't multiple connections be monitored simultaneously?
**A**: A single RF interface can only monitor one channel at a time. Time division multiplexing is needed to switch between connections.

### Q2: How to handle connection event conflicts?
**A**: Implement priority scheduling algorithm, or add hardware (use multiple Sniffer devices).

### Q3: How to recover after losing synchronization?
**A**: Monitor known channels, recapture packets by matching Access Address, use MIC (Message Integrity Check) to verify synchronization.

### Q4: Can encrypted connections be monitored?
**A**: Sniffer can capture encrypted packets but cannot decrypt content (unless keys are known). Metadata and timing information can be analyzed.

## Example Application Scenarios

1. **Protocol Debugging**: Verify correctness of Bluetooth protocol implementation
2. **Performance Analysis**: Measure connection interval, latency, throughput
3. **Fault Diagnosis**: Analyze connection drops, reconnections, etc.
4. **Interoperability Testing**: Verify compatibility between different vendor devices

## License and Disclaimer

This document is for educational and research purposes only. When using Sniffer functionality to monitor Bluetooth communications, please comply with local laws, regulations, and privacy protection requirements.

---

**Author**: sample.dotblog  
**Last Updated**: 2026-01-29  
**Version**: 1.0
