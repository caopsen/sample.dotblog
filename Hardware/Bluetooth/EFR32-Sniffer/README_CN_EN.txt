文档说明 / Documentation Guide
================================

本目录包含关于 EFR32BG24A010F1024IM40 芯片蓝牙 Sniffer 功能的完整文档。
This directory contains complete documentation for EFR32BG24A010F1024IM40 Bluetooth Sniffer functionality.

文档结构 / Documentation Structure
----------------------------------

1. README.md (中文)
   - 主要文档，详细说明 Sniffer 工作原理和实现方法
   - Main documentation explaining Sniffer principles and implementation

2. README.en.md (English)
   - README.md 的英文翻译版本
   - English translation of README.md

3. Documentation/Technical-Guide.md (中文)
   - 深入技术实现指南
   - 包含硬件配置、软件架构、协议解析等详细内容
   - In-depth technical implementation guide
   - Covers hardware configuration, software architecture, protocol parsing, etc.

4. Documentation/Quick-Start.md (中文)
   - 快速入门指南
   - 包含环境设置、项目创建、基础代码实现
   - Quick start guide
   - Covers environment setup, project creation, basic code implementation

核心问题解答 / Core Questions Answered
--------------------------------------

Q: EFR32BG24 芯片的 Sniffer 功能如何监听两个连接设备？
A: 通过以下机制：
   1. 捕获连接建立时的 CONNECT_REQ 数据包
   2. 提取每个连接的访问地址和跳频参数
   3. 使用时分复用在多个连接间切换
   4. 维护独立的连接上下文状态

Q: How does EFR32BG24's Sniffer monitor two connected devices?
A: Through the following mechanism:
   1. Capture CONNECT_REQ packets during connection establishment
   2. Extract access address and hopping parameters for each connection
   3. Use time division multiplexing to switch between connections
   4. Maintain independent connection context states

Q: 如何区分监听的是哪个设备？
A: 通过以下方式：
   1. 访问地址 (Access Address) - 每个连接的唯一 32 位标识符
   2. 蓝牙设备地址 (BD_ADDR) - 设备的 MAC 地址
   3. 连接上下文管理 - 为每个连接维护独立的状态结构
   4. 数据包元数据 - 时间戳、信道、RSSI 标记

Q: How to distinguish which device is being monitored?
A: Through the following methods:
   1. Access Address - Unique 32-bit identifier for each connection
   2. Bluetooth Device Address (BD_ADDR) - Device MAC address
   3. Connection Context Management - Maintain separate state structures
   4. Packet Metadata - Timestamp, channel, RSSI tagging

使用建议 / Usage Recommendations
--------------------------------

1. 初学者 / Beginners
   - 先阅读 README.md 或 README.en.md 了解基本原理
   - 然后参考 Quick-Start.md 进行实践
   - Read README.md or README.en.md to understand basic principles first
   - Then refer to Quick-Start.md for practice

2. 高级开发者 / Advanced Developers
   - 直接参考 Technical-Guide.md 获取详细实现细节
   - 根据实际需求修改示例代码
   - Refer directly to Technical-Guide.md for detailed implementation
   - Modify example code according to actual requirements

注意事项 / Important Notes
--------------------------

1. 文档中的代码示例为教学目的，部分函数需要完整实现
   Code examples in documentation are for educational purposes, some functions need complete implementation

2. 实际应用需要根据具体硬件和软件环境进行适配
   Actual applications need adaptation based on specific hardware and software environment

3. 使用 Sniffer 功能请遵守当地法律法规和隐私保护规定
   When using Sniffer functionality, comply with local laws and privacy protection regulations

版权信息 / Copyright
--------------------

Author: sample.dotblog
Date: 2026-01-29
Version: 1.0
License: Educational and research purposes only
