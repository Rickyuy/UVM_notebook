# Virtual Interface的介绍

## 1、为什么要使用interface

例如在driver中等待时钟事件，我们可以使用该语句(@posedge top.clk)，给DUT中输入端口赋值，我们可以使用该语句(top.rx_dv<=1'b1)，这些都是使用的绝对路径，绝对路径的使用大大减弱了验证平台的可移植性。如果clk信号的层次从top.clk变成了top.clk_inst.clk，那么就需要对driver中相关代码做大量的修改。

避免绝对路径的一个方法就是使用宏：
```
`define TOP top_tb
task my_driver::main_phase(uvm_phase phase);
    phase.raise_objection(this);
    `uvm_info("my_driver", "main_phase is called", UVM_LOW);
    `TOP.rxd <= 8'b0;
    `TOP.rx_dv <= 1'b0;
    while(!`TOP.rst_n)
        @(posedge `TOP.clk);
    for(int i = 0; i < 256; i++)begin
        @(posedge `TOP.clk);
        `TOP.rxd <= $urandom_range(0, 255);
        `TOP.rx_dv <= 1'b1;
        `uvm_info("my_driver", "data is drived", UVM_LOW);
    end
    @(posedge `TOP.clk);
    `TOP.rx_dv <= 1'b0;
    phase.drop_objection(this);
endtask
```

这样，当路径修改时，只需要修改宏的定义就可以了。但是加入clk的路径变为了top_tb.clk_inst.clk，而rst_n的路径变为了top_tb.rst_inst.rst_n，那么单纯地修改宏定义就无法起到作用。

避免绝对路径的另一种方法就是使用interface。在SystemVerilog中使用interface来连接验证平台与DUT的端口。

## 2、接口的定义

```
文件：src/ch2/section2.2/2.2.4/my_if.sv
4 interface my_if(input clk, input rst_n);
5
6 logic [7:0] data;
7 logic valid;
8 endinterface
```

定义了interface后，在top_tb中实例化DUT时，就可以直接使用：
```
文件：src/ch2/section2.2/2.2.4/top_tb.sv
17 my_if input_if(clk, rst_n);
18 my_if output_if(clk, rst_n);
19
20 dut my_dut(.clk(clk),
21 .rst_n(rst_n),
22 .rxd(input_if.data),
23 .rx_dv(input_if.valid),
24 .txd(output_if.data),
25 .tx_en(output_if.valid));
```