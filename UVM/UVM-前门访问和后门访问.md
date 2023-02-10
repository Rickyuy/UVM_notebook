# UVM-前门访问和后门访问

## 1、前门访问
通过寄存器配置总线SPI(如APB协议、OCP协议、I2C协议)来对DUT进行操作，前门访问操作只有两种方式：读和写操作

- 第一种uvm_reg::read()/write()，在传递时，用户需要将参数path指定为UVM_FRONTDOOR。除了status和value两个参数需要传入，其他参数可采用默认值。
```markdown
    1.寄存器模型提供的方法
    //reg_model寄存器模型实例名，reg寄存器实例名
    reg_model.reg.write(status, value, UVM_FRONTDOOR, .parent(this));
    reg_model.reg.read(status, value, UVM_FRONTDOOR, .parent(this));
```
- 第二种uvm_reg_sequence::read_reg()/write_reg(),在使用时，也要将path指定为UVM_FRONTDOOR
```markdown
    2.uvm_reg_sequence预定义方法
    read_reg(rgm.ss, status, data, UVM_FRONTDOOR);
    write_reg(rgm.ss, status, data, UVM_FRONTDOOR);
```
## 2、后门访问
与前门访问的操作相对，从广义上讲所有不通过DUT的总线而对DUT内部的寄存器或者存储器进行存取的操作都是后门访问，利用UVM DPI(uvm_hdl_read()、uvm_hdl_deposit()),而不通过物理总线访问

a. 确保寄存器模型在建立时将各个寄存器映射到DUT一侧的HDL路径
```markdown
add_hdl_path("reg_backdoor_access.dut");
chnl0_ctrl_reg.add_hdll_path_slice($sformatf("regs[%0d]", `SLVO_RW_REG), 0, 32);
lock_model();
```
- 通过uvm_reg_block::add_hdl_path()将寄存器模型关联到DUT一端
- 通过uvm_reg::add_hdl_path_slice完成将寄存器模型各个寄存器成员与HDL一侧的地址映射
- lock_model()函数结尾，结束地址映射关系，保证模型不会被其他用户修改

b. 寄存器模型完成HDL路径映射后，利用uvm_reg或uvm_reg_sequence自带的方法进行后门访问

- uvm_reg::read()/write(), 在调用该方法时设置UVM_BACKDOOR的访问方式
- uvm_reg_sequence::read_reg()/write_reg(), 在调用该方法时设置UVM_BACKDOOR的访问方式
- uvm_reg::peek()/poke()两个方法，分别对应了读取寄存器(peek)和修改寄存器(poke)两种操作，本身只针对后门访问，所以无需设置UVM_BACKDOOR
```markdown
1. 寄存器模型提供的方法read()/write()
rgm.ctrl.read(status, data, UVM_BACKDOOR, .parent(this));
rgm.ctrl.write(status, 'h11, UVM_BACKDOOR, .parent(this));
2. uvm_reg_sequence预定义方法read_reg()/write_reg()
read_reg(rgm.ss, status, data, UVM_BACKDOOR);
write_reg(rgm.ss, status, 'h22, UVM_BACKDOOR);
3. 寄存器模型提供的方法peek()/poke()
rgm.ctrl.peek(status, data, .parent(this));
rgm.ctrl.poke(status, 'h22, .parent(this));
```
|前门访问|后门访问|
| ------ | ------ |
|通过总线协议访问需要耗时，且在总线访问结束时才能结束前门访问|通过UVM DPI关联硬件寄存器信号路径，直接读取或修改硬件，不需要访问时间，零时刻响应|
|一般读写只能按字(word)读写，无法直接读写寄存器域|可以对寄存器或寄存器域直接做读写|
|依靠监测总线来对寄存器模型内容做预测|依靠auto prediction方式自动对寄存器内容做预测|
|正确反应了时序关系|不受硬件时序控制，对硬件做的后门访问可能发生时序冲突|
|不受总线时序功能影响|通过总线协议，可以有效捕捉总线错误，继而验证总线访问路径|