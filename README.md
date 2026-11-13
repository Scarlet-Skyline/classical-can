实现classical-CAN与主机的双向桥接

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

3. 并发与缓冲
任务分工：基于 Embassy 异步运行时，通过 join/join4 并发调度 6 个任务：device.run（USB底层）、control_port（命令解析与收发）、uart_port（UART1透传）、dap.run（调试接口）、can_rx_task（CAN接收）、can_tx_task（CAN发送）。

队列容量：使用 embassy_sync::channel::Channel 提供有界缓冲。接收通道 CAN_RX_CHANNEL 容量为 8；发送通道 CAN_TX_CHANNEL 容量为 4。

满队列处理机制：

接收满：总线数据过快，主机处理慢时，can_rx_task 中的 try_send 返回失败，直接丢弃当前帧。这保证了 MCU 绝不会被 USB 端阻塞，CAN 硬件不会溢出报错。

发送满：主机连续高频发送命令时，control_port 中的 try_send 失败，立即通过 USB 回复 ERR BUSY\n，不阻塞主机进程。

中断/DMA：FDCAN 接收和发送均依赖底层硬件 FDCAN1_IT0/IT1 中断 驱动（非 DMA）。UART 端口使用了 BufferedUart，底层由 USART1 中断 配合静态环形缓冲区进行数据搬运。

