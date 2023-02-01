# Ack Nak机制
Ack/Nak是一种由硬件实现的，完全自动的机制，目的是保证TLP有效可靠地传输。Ack DLLP用于确认TLP被成功接收，Nak DLLP则用于表明TLP传输中遇到了错误。

![Ack_Nak机制1](./image/Ack_Nak%E6%9C%BA%E5%88%B61.png "Ack_Nak机制1")

如图所示，发送方会对每一个TLP在Replay Buffer中做备份，直到其接收来自接收方的Ack DLLP，确认该DLP已经成功的被接收,才会删除这个备份。如果接收方发现TLP存在错误，则会向发送方发送Nak DLLP,然后发送方会从Replay Buffer中取出数据，重新发送该TLP。

Ack/Nak机制内部的详细结构图如下图所示：

![Ack_Nak机制内部详细结构图](./image/Ack_Nak%E6%9C%BA%E5%88%B6%E5%86%85%E9%83%A8%E8%AF%A6%E7%BB%86%E7%BB%93%E6%9E%84%E5%9B%BE.png "Ack_Nak机制内部详细结构图")

下面对图中的各个Elements分别做一个简单的介绍：

- 发送端的Elements：

![发送端Elements](./image/%E5%8F%91%E9%80%81%E7%AB%AFElements.png "发送端Elements")

1. NEXT_TRANSMIT_SEQ Counter，即NTS计数器，是一个12位的计数器。当数据链路层处于DL-Down状态或者复位时，该计数器会被初始化位0.该计数器只会执行加一操作，也就是说当其到达最大值4095时，在进行加一操作则会变成0(Roll Over)。该计数器用于产生下一个待发送的TLP的序列号(Sequence Number)。每一个序列号都是与一个TLP所唯一对应的，可以说这个序列号是整个Ack/Nak机制的关键。
2. LCRC Generator，LCRC产生器用于产生一个32位的CRC值，其作用于整个TLP和其对应的序列号。
3. Replay Buffer，是Mindshare书中的叫法，在PCIe Spec中，这个Buffer的名称叫做Retry Buffer。Replay Buffer中按照传输顺序，存储了整个TLP、序列号和LCRC。当发送端收到来自接收端的Ack DLLP时，会将Buffer中相应的TLP(包括对应的序列号和LCRC)移除；如果接收到的是Nak DLLP，则会将Buffer中响应的TLP(包括对应的序列号和LCRC)重新发送给接收端。
4. REPLAY_TIMER Count，REPLAY_TIMER是一种看门狗定时器，当该定时器溢出，则表明发送端已经发送了一个或者多个TLP，但是并未收到来自接收端的应答信号(Ack/Nak)。此时，发送端会将Replay Buffer中的TLP重新发送，并将看门狗定时器重启。只要发送端发送了任何TLP，该定时器便会启动，在接收到来自接收端的应答信号之前都会持续地运行。当收到应答信号之后，定时器会被立即清零。此时如果Replay Buffer仍然有TLP(表明还有TLP被发送，但是仍未得到应答)，定时器又会被立即重新启动。如果Buffer中是空的，则定时器不会被重新启动，直到新的TLP被发送。
5. REPLAY_NUM Count，这是一个2位的计数器，用于记录同一个TLP发送失败的次数，当其值从11b变为00b时(溢出了，表示尝试发送某个TLP失败了4次)，数据链路层会自动地强制物理层重新进行链路训练(即LTSSM进入Recovery状态)。当完成链路训练之后，便会重新发送之前发送失败的TLP。当发送端接收到来自接收端的Nak DLLP或者发送端的看门狗定时器(REPLAY_TIMER)溢出时，该计数器都会被加一；当接收到Ack DLLP时，该计数器则会被清零。
6. ACKD_SEQ Register，ACKD_SEQ寄存器用于存储最近接收到的Ack或者Nak DLLP中的序列号。当复位或者数据链路层处于无效状态时，该寄存器会被初始化为全1。
7. DLLP CRC Check，接收端在接收到来自发送端的DLLP后，首先会检查其DLLP CRC，如果发现有错误，则会直接将其丢弃，认为其是无效的DLLP。
   
- Replay Buffer中按照传输顺序，存储了整个TLP、序列号和LCRC，如下图所示：

![Retry_Buffer结构图](./image/Retry_Buffer%E7%BB%93%E6%9E%84%E5%9B%BE.png "Retry_Buffer结构图")

- 接收端的Elements：
  
![接收端Elements](./image/%E6%8E%A5%E6%94%B6%E7%AB%AFElements.png "接收端Elements")

1.LCRC Error Check，顾名思义，LCRC Error Check用于检查收到的TLP是否存在错误。如果存在错误，则将对应的TLP直接丢弃，然后产生一个Nak DLLP发送给发送端，让其重新发送该TLP。
2.NEXT_RCV_SEQ Counter，是一个12位的计数器，即Next Receive Sequence Number，其值为已经成功接收的TLP的序列号加1.主要用于检查当前接收到的TLP是不是应该接收到的TLP。

如果NEXT_RCV_SEQ和当前接收到的TLP中的序列号的值相等，则认为这是一个有效的TLP，但是接收端并不会立即向发送端发送Ack DLLP，而是等到AckNak_LATENCY_TIMER溢出时才向发送端发送Ack DLLP。也就是说，一个Ack DLLP可能会对应多个TLP，接收端不会每成功接收到一个TLP便向发送端发送Ack DLLP。

如果当前接收到的TLP中的序列号小于NEXT_RCV_SEQ(且差值不超过2048)，则认为该TLP之前已经成功发送过了，此次是重复发送。需要注意的是，PCIe Spec认为重复发送并不是一个错误，只是直接将该TLP丢弃，并没有Nak或者Error Reporting，但是会返回一个包含有上一次成功接收的TLP的序列号(NRS-1)的Ack DLLP给发送端。

如果当前接收到的TLP的序列号大于NEXT_RCV_SEQ，表明传输过程中漏掉了一些TLP。此时，接收端会返回Nak DLLP，并直接丢弃该TLP。
3.NAK_SCHEDULED Flag，是一个标志位，当接收端产生Nak DLLP时，该标志位会被置位。当接收端成功接收到有效的TLP时，该标志位会被清零。需要特别注意的是，当该标志位处于置位状态时，接收端不应产生其他的Nak DLLP.
4.AckNak_LATENCY_TIMER，定时器会在接收端成功接收到有效的TLP，且并未向发送端返回Ack DLLP之前运行。当AckNak_LATENCY_TIMER定时器溢出时，接收端会立即向发送端返回Ack DLLP(携带的序列号为NRS-1，即一个Ack对应多个有效的TLP)。无论接收端返回Ack还是Nak，该定时器都会被复位，但是只有当接收端再次收到有效的TLP时，该定时器才会被重新启动。
5.Ack/Nak Generator，产生Ack或者Nak DLLP。
