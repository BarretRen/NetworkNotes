# wifi 工作状态

通常情况下，802.11 设备一共会有 4 个工作状态：

- Sleep（休眠模式）：节点会关闭发送和接收模块进行休眠，从而能耗最低。
- Rx Idle（接收空闲状态）：节点对信道进行监听，但并未真实接收数据帧。
- Rx（接收状态）：节点监听到数据帧，并对其进行接收。
- Tx（发送状态）：节点发送数据帧

# PSM 节能模式

- 节点发送数据，用 Rx Idle 来监听信道的，从而如果没有数据发送，那么就不进行监听，从而就可以减少 Rx Idle 的持续时间了，这个还是很容易做到的
- 节点接收数据, 使用**缓存机制和被动请求机制**让节点去请求接收接收 AP 发来的数据
  - AP 可以通过 **TIM** 字段主动告知 STA 有数据被缓存, 需要 STA 去主动接收
  - AP 发给 sta 的帧中的**more data**flag 可以标识该节点是否还有缓存数据

## AP 缓存机制

当 AP 从外网接收到要转发给节点的数据后，会将以 MSDU 的形式（即 MAC 层的数据帧）缓存在队列中（仅对节能模式下的节点数据进行缓存），并不直接发送给节点

## PS-POLL 机制

![Alt text](4_powersave.assets/image-2.png)

- 节点想要获取下行数据，那么节点需要主动跟 AP 请求数据(PS-Poll 帧)
- AP 接收到该帧后，会检查缓存区是否有对应该节点的缓存，如果有就会从缓存区中调出对应该节点的缓存数据，并进行下发
- 如果没有则反馈一个 NULL 帧（既空数据）

## DTIM 保活睡眠

TIM：每一个 Beacon 的帧中都有一个**TIM**，它主要用来**由 AP 通告它管辖下的哪个 STA 有数据现在缓存在 AP 中**，

- TIM 中包含一个 Bitmap，它最大是 251 个字节，每一位映射一个 STA，当为 1 时表示该位对应的 STA 有数据在 AP 中. 结构如下
  ![Alt text](4_powersave.assets/image-1.png)
- STA 收到与自己关联的 TIM 就要发送 PS-POLL 帧去请求缓存的数据

DTIM 是 TIM 的特殊情况，**当发送几个 TIM 之后，就要发送一个 DTIM. 一旦 AP 发送了 DTIM, STA 就必须处于清醒，因为广播或组播无重发机制**，不醒来数据就收不到了。

TIM/DTIM 报文格式如下:
![Alt text](4_powersave.assets/image.png)

- DTIM Count： 当前的 DTIM 值
  - 当 DTIM Count 为 0 表示当前的 TIM 是一个 DTIM
- DTIM Period： 在 AP 中配置的 DTIM 周期
  - 如果 DTIM 周期设置为 1 表示每一个 beacon 都是 DTIM beacon

因此针对 DTIM, STA 可以设置**保活睡眠**(也叫 dtim power save)来减少功耗:
**STA 通过 listen interval 来设置监听 beacon 的间隔: 可以设置成和 DTIM 的间隔一样，或者是 DTIM Period 的倍数。AP 通过 assoc req 帧得知 interval, 会在节点的苏醒时在 TIM 中为其指示缓存状态. 这样一来，每次 DTIM 到来的时候，既可以接收广播数据，也可以接收单播的数据**。
