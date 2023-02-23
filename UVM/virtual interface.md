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

因为driver是一个类，在类中不能通过声明接口再通过赋值的形式使用接口，只有在类似top_tb这样的模块(module)中才可以。在类中使用的是virtual interface：
```
文件：src/ch2/section2.2/2.2.4/my_driver.sv
3 class my_driver extends uvm_driver;
4
5 virtual my_if vif;
```

在声明了vif后，就可以在main_phase中使用如下方式驱动其中的信号：
```
文件：src/ch2/section2.2/2.2.4/my_driver.sv
23 task my_driver::main_phase(uvm_phase phase);
24 phase.raise_objection(this);
25 `uvm_info("my_driver", "main_phase is called", UVM_LOW);
26 vif.data <= 8'b0;
27 vif.valid <= 1'b0;
28 while(!vif.rst_n)
29 @(posedge vif.clk);
30 for(int i = 0; i < 256; i++)begin
31 @(posedge vif.clk);
32 vif.data <= $urandom_range(0, 255);
33 vif.valid <= 1'b1;
34 `uvm_info("my_driver", "data is drived", UVM_LOW);
35 end
36 @(posedge vif.clk);
37 vif.valid <= 1'b0;
38 phase.drop_objection(this);
39 endtask
```
可以清楚看到，代码中绝对路径已经消除了，大大提高了代码的可移植性和重用性。

## 3、如何把module中的interface与class中的virtual interface对应起来

**问题点：在top_tb中不可能像引用my_dut一样直接引用my_driver中的变量：top_tb.my_dut.xxx是可以的，但是top_tb.my_driver.xxx是不可以的。这个问题的终极原因在于UVM通过run_test语句实例化了一个脱离top_tb层次结构的实例，建立了一个新的层次结构。**

对于这种脱离了top_tb层次结构，同时又希望在top_tb中对其进行某些操作的实例，UVM引进了**config_db机制**。在config_db机制中，分为set和get两步操作。

在top_tb中执行set操作：
```
文件：src/ch2/section2.2/2.2.4/top_tb.sv
44 initial begin
45 uvm_config_db#(virtual my_if)::set(null, "uvm_test_top", "vif", input_if);
46 end
```
在my_driver中执行get操作：
```
文件：src/ch2/section2.2/2.2.4/my_driver.sv
13 virtual function void build_phase(uvm_phase phase);
14 super.build_phase(phase);
15 `uvm_info("my_driver", "build_phase is called", UVM_LOW);
16 if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
17 `uvm_fatal("my_driver", "virtual interface must be set for vif!!!")
18 endfunction
```

这里引入了build_phase。与main_phase一样，build_phase也是UVM中内建的一个phase。当UVM启动后，会自动执行build_phase。build_phase在new函数之后main_phase之前执行。在build_phase中主要通过config_db的set和get操作来传递一些数据，以及实例化成员变量等。需要注意的是，这里需要加入super.build_phase语句，因为在其父类的build_phase中执行了一些必要的操作，这里必须显式地调用并执行它。build_phase与main_phase不同的一点在于，build_phase是一个函数phase，而main_phase是一个任务phase，build_phase是不消耗仿真时间的。build_phase总是在仿真时间（$time函数打印出的时间）为0时执行。

在build_phase中出现了uvm_fatal宏，uvm_fatal宏是一个类似于uvm_info的宏，但是它只有两个参数，这两个参数与uvm_info宏的前两个参数的意义完全一样。与uvm_info宏不同的是，当它打印第二个参数所示的信息后，会直接调用Verilog的finish函数来结束仿真。uvm_fatal的出现表示验证平台出现了重大问题而无法继续下去，必须停止仿真并做相应的检查。所以对于uvm_fatal来说，uvm_info中出现的第三个参数的冗余度级别是完全没有意义的，只要是uvm_fatal打印的信息，就一定是非常关键的，所以无需设置第三个参数。

config_db的set和get函数都有四个参数，这两个函数的第三个参数必须完全一致。set函数的第四个参数表示要将哪个interface通过config_db传递给my_driver，get函数的第四个参数表示把得到的interface传递给哪个my_driver的成员变量。set函数的第二个参数表示的是路径索引。在top_tb中通过run_test创建了一个my_driver的实例，那么这个实例的名字是什么呢？答案是uvm_test_top：UVM通过run_test语句创建一个名字为uvm_test_top的实例。

> 如果要向my_driver的var变量传递一个int类型的数据，那么可以使用如下方式：
>```
>initial begin
>   uvm_config_db#(int)::set(null, "uvm_test_top", "var", 100);
>end
>```
>而在my_driver中应该使用如下方式：
>```
>class my_driver extends uvm_driver;
>   int var;
>   virtual function void build_phase(uvm_phase phase);
>   super.build_phase(phase);
>   `uvm_info("my_driver", "build_phase is called", UVM_LOW);
>    if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
>   `uvm_fatal("my_driver", "virtual interface must be set for vif!!!")
>   if(!uvm_config_db#(int)::get(this, "", "var", var))
>   `uvm_fatal("my_driver", "var must be set!!!")
>endfunction
>```
