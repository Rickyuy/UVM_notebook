# PCIE时序
![PCIE时序图](./image/pcie%E6%97%B6%E5%BA%8F%E5%9B%BE.png "PCIE时序图")
- 在第一个时钟上升沿，FRAME#和IRDY#都为inactive，表明总线当前处于空闲状态。与此同时，某设备的GNT#信号处于active，表明总线总裁器已经选定当前设备为下一个initiator(可以理解为主机)。
- 在第二个时钟上升沿，FRAME#被initiator拉低，表明新的事务（Transaction)已经开始。与此同时，地址和命令被依次发送到AD上，总线上面的所有其他设备（从机）都会锁存这些信息，并检查地址和命令是否与自己匹配。
- 在第三个时钟上升沿，IRDY#处于active状态，表明主机准备就绪，可以接收数据了。AD信号上的旋转的箭头表示AD信号目前属于三态状态（属于输入和输出的转换状态，即Turn-around cycle。需要注意的是，此时TRDY#应当处于inactive状态，以保证Turn-around cycle顺利进行。
- 在第四个时钟上升沿，PCI总线上的某个从机确认身份，并依次将DEVSEL#信号和TRDY#信号拉低，并将相应的数据输出到AD上。此时，FRAME#信号为active状态，表明这并不是最后一个数据。
- 在第五个时钟上升沿，TRDY#处于inactive，表明从机尚未就绪，因此所有的操作暂缓一个时钟周期。PCI总线最多允许8个这样的Wait State。
- 第六个时钟上升沿，从机向主机发送第二个数据。此时，FRAME#信号依旧为active状态，表明这并不是最后一个数据。
- 第七个时钟上升沿，IRDY#处于inactive状态，表明主机尚未就绪，再次插入一个Wait State。但是此时从机依旧可以向AD上发送数据。
- 在第八个时钟上升沿，AD上的第三个数据被发送至主机，由于此时FRAME#信号被拉高，即inactive，表明这是本次事务（Transaction）的最后一个数据。此后，所有的控制信号均被拉高，处于inactive状态，AD、FRAME#和C/BE#处于三态状态。