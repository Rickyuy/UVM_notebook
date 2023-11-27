# I2C_VIP环境配置
一、环境架构

bfms(Bus Functional Models)总线功能模型中存放I2C的Basic Env如：i2c_basic_env.sv、
i2c_scoreboard.sv、i2c_virtual_sequencer.sv、
i2c_env_pkg.svh

在Testbench中存放test和seq文件，如i2c_base_test.sv、i2c_mst_directed_sequence.sv、i2c_test_pkg.svh

在调用的时候import i2c_test_pkg即可。

二、环境属性

下面记录一下S家I2C VIP几个需要注意的属性配置

①配置I2C总线的速度：cfg.set_bus_speed(XXX);

②VIP作Slv时配置I2C的slave address：cfg.slave_cfg/mst_cfg[x].slave_address = XXX;

③配置成slv address10bit模式：cfg.slave_cfg/mst_cfg[x].enable_10bit_addr = XXX;

④提高uvm_info等级进行debug：i2c_env[x].set_report_verbosity_level(UVM_HIGH);

⑤取消VIP内部的pullup：XXX.XX_i2c_if.enable_pull_up_resistor = 1'b0;

三、Transaction属性配置

master transaction配置：

①cmd:I2C_READ/I2C_WRITE IIC的读写指令（当VIP作MST时需要配置）

②addr：当VIP为MST配置发送的SLV address

③data.size() = XXX;配置数据长度

④data[X] = XXX;配置数据（当cmd为I2C_WRITE是不需要配置）

⑤sr_or_p_gen: 

值为1的时候 --> 配置Master产生repeated Start信号

值为0的时候 --> 配置Master产生Stop信号

⑥send_start_byte: 配置Master发送Start Byte

⑦m_device_id_gen_stop : 当Master需要设备ID的时候，根据帧格式的协议，在Slave设备Ack之后，Master需要产生一个重复的Start信号，而m_device_id_gen_stop配置成1之后会force一个stop信号来代替start信号。

slave transaction配置：

①nack_addr: 

如果配置为1，Slave将回复一个NACK给主设备

如果配置为0，Slave将回复一个ACK给主设备

②data = new[X] :为数据申请一段空间，即datasize

③data[X] = XXX; 配置数据

