实现classical-CAN与主机的双向桥接

修改了：protocol.rs (定义数据格式和转换规则,定义如 struct CanFrame 或 struct UsbPacket，并包含将 CAN 数据“打包”成 USB 格式，或者将 USB 数据“解包”成 CAN 格式的函数。负责处理帧 ID、数据长度（DLC）等细节) , main.rs(程序入口，实现双向桥接逻辑：启动一个循环或异步任务。一方面监听 CAN 总线，一旦有数据就打包通过 USB 发给主机（要通过usb-can分析仪，此处用DMTool）；另一方面监听 USB，一旦主机发来指令，就解析并转换成 CAN 帧发到总线上。控制着数据流向。), Cargo.toml (加上用于异步处理的 embassy 框架)

1. 数据路径
CAN 数据到达主机（接收）：
外部 CAN 节点（如达妙电机或 DMTool 模拟发送）向总线发送帧 -> MCU 的 FDCAN1 硬件通过 RX 引脚（PD0）接收 -> 触发中断 -> can_rx_task 被唤醒读取帧 -> 将 Frame 数据推入容量为 8 的 CAN_RX_CHANNEL（通道队列）-> control_port 任务监听到队列有数据，取出并格式化为 RX <ID> <Data>\n 文本 -> 通过 USB CDC 虚拟串口（Control Port）发送给电脑主机。

主机请求变成 CAN 帧（发送）：
电脑主机通过 USB CDC 发送命令 CAN TX <ID> <Data>\n -> control_port 任务读取字节流并用 Line 结构拼接识别 -> 解析出 ID 和 Hex 数据 -> 构造标准 Frame 结构体 -> 放入容量为 4 的 CAN_TX_CHANNEL 并立刻回复主机 OK QUEUED\n -> can_tx_task 取出该帧 -> 调用 FDCAN1 的 write 方法通过 TX 引脚（PD1）发送到 CAN 总线 -> DMTool 或电机接收。

2. 主机协议
字段与边界：采用 文本协议（ASCII）。所有命令和回复均以换行符 \n 作为帧边界（定界符）。单行长度不超过 96 字节，USB 每次传输不超过 64 字节（单包）。但是，classical CAN 物理层限制：数据区最多 8 字节（代码里用 [u8; 8] 装）。单行长度最大为96字节是支持多包拼接昉溢出。

发送请求（主机 -> MCU）：

格式：CAN TX <11位Hex ID> <Hex数据>\n（数据可以是 0-8 字节，空数据用 - 表示）。

示例：CAN TX 123 11223344\n

接收消息（MCU -> 主机）：

格式：RX <ID> <Hex数据>\n（MCU 主动推送，主机只需按行读取）。

示例：RX 123 11223344\n

测试链路配合：将 DMTool 波特率设为 1Mbps，串口助手连接 Control Port（波特率任意，此处为 921600），在 DMTool can界面发 11 22 33 44 （数据长度为4字节，没超过8字节） ，帧类型：数据帧，帧格式：标准帧，发送后显示 “发送成功”。

$\color{red}{\text{
注意：这部分结合了本次测试的实际情况，对硬件测试做出详细说明，
    重点包括了双向验收的不同信号传输渠道/需注意的硬件连接方式/预期测试结果等
    测试使用DMTool USB-CAN分析仪及其上位机软件
}}$

$\color{red}{\text{
1. 主机USB接DMTool USB接口，DMTool的输出有CAN总线和UART（USART）两种信号，想双向测试都得接上。UART 接 MCU UART，CAN 接 MCU CAN。
   接上了CAN信号渠道，通过DMTool发出的CAN总线信号才能被 MCU CAN底层硬件所接受，通过MCU芯片处理，生成 RX <ID> <Hex数据>\n 信号发给
   主机，但是只有连上了MCU和DMTool的UART信号渠道才能在界面读到这条消息。
2. 主机--->MCU
   调到DMTool的“串口”界面，下方白框区域发送CAN TX <11位Hex ID> <Hex数据>\n，那如何查看呢？下一步调到 “CAN” 界面进行查看，会显示从
   主机发来的消息
3. MCU--->主机
   这一步不用那么繁琐的输入 “CAN TX XXXXXXXX”等内容，直接调到"CAN"界面，下方有专门的DLC/数据内容/帧ID设置。设置好后点击“发送”，
   会显示“发送成功”。此时表示波特率配置/接口选择等硬件测试环节没问题，MCU CAN硬件部分已接受这个信号。如果MCU软件没问题，调回“串口”
   界面，会看到程序中设计好的 RX <ID> <Hex数据>\n
4. DMTool和MCU之间两种信号的传输的切换由上位机软件中的不同接口、波特率决定，使用时不仅仅是切换窗口就行了，还要注意切换接口和波特率。
   如果USB-CAN分析仪没有UART信号接口，则要用USB-TTL线连接MCU UART和主机，并且下载串口助手，进行测试指令 “CAN TX <11位Hex ID>
   <Hex数据>\n” 的输入和CAN总线的监听。但是分析仪一般是两种信号都有的。
}}$   

6. 并发与缓冲
任务分工：基于 Embassy 异步运行时，通过 join/join4 并发调度 6 个任务：device.run（USB底层）、control_port（命令解析与收发）、uart_port（UART1透传）、dap.run（调试接口）、can_rx_task（CAN接收）、can_tx_task（CAN发送）。

队列容量：使用 embassy_sync::channel::Channel 提供有界缓冲。接收通道 CAN_RX_CHANNEL 容量为 8；发送通道 CAN_TX_CHANNEL 容量为 4。

满队列处理机制：

接收满：总线数据过快，主机处理慢时，can_rx_task 中的 try_send 返回失败，直接丢弃当前帧。这保证了 MCU 绝不会被 USB 端阻塞，CAN 硬件不会溢出报错。

发送满：主机连续高频发送命令时，control_port 中的 try_send 失败，立即通过 USB 回复 ERR BUSY\n，不阻塞主机进程。

中断/DMA：FDCAN 接收和发送均依赖底层硬件 FDCAN1_IT0/IT1 中断 驱动（非 DMA）。UART 端口使用了 BufferedUart，底层由 USART1 中断 配合静态环形缓冲区进行数据搬运。

