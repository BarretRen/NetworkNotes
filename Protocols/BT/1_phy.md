# 介绍

## 蓝牙发展历史

> 目前已经到了 5.4 版本, 可以在官网下载协议标准: https://www.bluetooth.com/zh-cn/specifications/specs/core-specification-5-4

![Alt text](1_phy.assets/image-2.png)

## 蓝牙分类

蓝牙用于短距离交换资料，形成个人局域网 PAN. 使用短波特高频 UHF 无线电波. 分为两种:
![Alt text](1_phy.assets/image-1.png)

- BR/EDR: 基础率/增强数据率, 以**点对点**拓扑结构创建一对一的设备通信
- BLE: 低耗能, 使用**点对点**(一对一)、**广播**(一对多)和**网格**(多对多)等多种拓扑结构

# 物理层

## 频段和信道划分

BLE**在 2.4 ~ 2.485GHz 的 ISM 频段上通信**, 占用 40 个射频信道，信道间隔 2MHz，中心频率为`2402+k*2MHz`。k=0,1,…,39.分为两类:
![Alt text](1_phy.assets/image-4.png)

- 广播信道: 37, 38, 39
  - <font color='red'>蓝牙 5.0 以后增加了第二广播信道，允许除 37、38、39 三个信道外的其它信道上进行广播</font>
- 数据信道: 其他

## 跳频和重传机制

由于上面频段内有 wifi 等各种设备的干扰, 蓝牙采用了**自适应跳频技术**, 避开干扰频段. 自适应跳频技术基本原理是根据信道状态的好坏，选择蓝牙通信信道。通常可以根据接收信号的 RSSI 和 BER 来判断。
![Alt text](1_phy.assets/image-5.png)

通过跳频机制，BLE 可以**有效降低碰撞几率，但这只是降低，而不是完全消除**。一旦环境中蓝牙或者是 2.4G 设备多了，碰撞几率还是会大大发生，这里就需要到 LL 层中另外的一套**重传机制**了。

- 当碰撞发生后，数据会出现乱码丢包，设备会自行跳到下一个可用信道进行重传
- 要是新的可用新的又碰撞，那就再跳。直到重传成功
- 或者跳的次数超过了连接参数中的 latency 和 timeout 指标，这时候就会判定为设备断连

通过这样的一套跳频和重传机制，BLE 能保证连接事件中数据要么成功发送对端，要么就直接断连。

## 4 种工作模式

![Alt text](1_phy.assets/image-6.png)

- LE 1M sym/s：非编码模式，S=2 映射编码模式，S=8 映射编码模式。
- LE 2M sym/s：非编码模式。

# 链路层

## 射频状态

LL 可以控制设备的射频状态，让设备处于这五种状态之一:

- Standby：默认状态，不进行收发。
- Advertising：广播状态，在 3 个广播信道广播数据包，同时监听和回复扫描者发送的扫描数据包。
- Scanning：扫描状态，在 3 个广播信息监听广播数据包，同时发送扫描数据包。
- Initiating：初始化状态，在广播信道监听广播数据包，从而发起连接。
- Connection：连接状态

## 帧格式

![Alt text](1_phy.assets/image-7.png)

- Access Address 用来标示接收者 ID 或者空中包身份
  - 广播包 Access Address 固定为 **0x8E89BED6**，广播包只能在广播信道（channel）上传输
  - 数据包 Access Address 为一个 **32bit 的随机值**，由 Initiator 生成。数据包只在数据信道上传输. 每建立一次连接，重新生成一次 Access address
- 无论是广播包还是数据包, payload 部分各不相同, 需要根据 PDU 类型解析

### 广播包

广播包 header:

- PDU type: 广播包的类型, 在**协议 core.pdf 的 Table 2.3: Advertising physical channel PDU header’s PDU Type field encoding 中定义**.
  - ADV_IND: 通用广播，包含广播数据和扫描响应数据
  - ADV_DIRECT_IND： 定向广播类型，为了尽可能快的连接，俗称**回连包**，发起者收到发给自己的定向广播报文之后，可以立即发送连接请求作为回应。定向广播，设备不能被主动扫描。此外，定 payload 中也不能带有其他附加数据。只能包含广播者的地址和发起者的地址
  - ADV_NONCONN_IND： 仅仅发送广播数据，而不想被扫描或者连接
  - ADV_SCAN_IND： 这种广播不能用于发起连接，但允许其他设备扫描该广播设备
- ChSel: 是否支持 LE channel 选择算法
  - #1 信道选择算法： ble 诞生以来就支持
  - #2 信道选择算法： ble 5.0 版本引入
- TxAdd: 广播设备的地址是否是随机的
- RxAdd: 目标设备的地址是否是随机的

有效数据部分：包含 N 个 AD Structure，每个 AD Structure 由 Length，AD Type 和 AD Data 组成:
![Alt text](1_phy.assets/image-3.png)

- Length：AD Type 和 AD Data 的长度
- AD Type：指示 AD Data 数据的含义, **在协议 Assigned_Number.pdf Common Data Types 定义**
- AD Data: 数据部分

广播包解析示例:
![Alt text](1_phy.assets/image-20.png)

### 数据包

数据包分为 l2cap header(length + cid)和 data 两部分, 其中 data 部分的内容根据 LLID 的不同含义不同:

- 数据包: <font color='red'>注意: 上层功能(比如 SMP)都是通过数据包传输, 再根据具体的要求填充和解析内容. 但在 LL 层看来都是数据包</font>
  - `LLID = 0b01`: LL data PDU, 空数据, 或非首个分片的 l2cap 包
  - `LLID = 0b10`: LL data PDU, 不分片的 l2cap 包, 或首个分片的 l2cap 包
- `LLID = 0b11`: LL control PDU, 控制 link layer 的连接, **opcode 在协议 core.pdf Table 2.20: LL Control PDU opcodes 中定义**, 格式如下:
  ![Alt text](1_phy.assets/image.png)

# 相关问题

## 蓝牙广播长度为什么是 31 字节

蓝牙规范规定链路层（Link Layer）的广播信道 PDU 最大为 37 字节。其中：

- 6 字节被用于广播设备地址（MAC Address）。
- 剩下的 31 字节 完全留给了 广播数据（ADV Data） 本身

**同理，scan response data 也是 31 字节长度。**

## 如何突破广播 31 字节限制

1. 使用扫描回应数据 (Scan Response Data) - 最常用、兼容性最好
   - 设备在主广播包（31 字节）中放置最核心、最紧急的信息（例如，设备 ID、UUID 等）。
   - 扫描设备（如手机）如果想知道更多信息，可以向广播设备发送一个 Scan Request。
   - 广播设备收到请求后，回复一个 Scan Response 包，其中包含附加数据（例如，完整的设备名称、自定义服务数据、传感器读数等）
1. 使用蓝牙 5 的扩展广播 (Extended Advertising) - 不支持蓝牙 5 之前版本
   - 设备在主广播信道发送一个非常短的“公告”，指示有扩展广播数据可用以及其在哪个信道（频率）上。
   - 扫描设备接收到这个公告后，如果在乎这些数据，会跳转到指定的次广播信道去监听一个或多个包含大量数据的扩展广播包

**扩展广播**允许在**次广播信道上使用更长的数据单元**(PDU)广播数据的大小不再局限于 31 字节，理论上最大可达到 1650 字节。

## BLE 广播间隔

广播间隔的单位是**0.625ms**(**扫描的间隔单位也是此值**)，在创建 adv activity 时要指定参数：

- `adv_intv_min`：表示每个信道上的最小广告间隔时间。最小值不能小于 20ms。它决定了设备在开始新的广告周期之前需要等待的最短时间
- `adv_intv_max`：表示每个信道上的最大广告间隔时间。最大值不能小于 20ms。它决定了设备在开始新的广告周期之前可以等待的最长时间

这两个参数共同决定了设备进行广告活动的时间间隔，从而影响设备发现和连接的效率。
首先在 primary channels 上使用 **ADV_EXT_IND**类型的广播，该类型的广播 PDU（具体参考 Vol 6, Part B）中就包括了如下信息：

- 接下来会使用哪个 secondary advertising channel 传输 AUX_ADV_IND
- 何时开始发送 AUX_ADV_IND 类型的广播

![alt text](1_phy.assets/image-8.png)
![alt text](1_phy.assets/image-9.png)

## BLE 连接间隔

连接间隔的单位是**1.25ms**，在创建 con activity 时要指定参数：

- fullcode：一个参数`con_interval`
- armino: 两个参数, 指定最大最小值
  - `intv_min`: ble 的最小通讯周期, 连续收发数据包时主机会选择此值, 不能小于 7.5ms
  - `intv_max`: ble 的最大通讯周期, 一般用于空闲状态, 不能小于 7.5ms
