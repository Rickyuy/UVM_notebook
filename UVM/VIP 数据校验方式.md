# VIP数据校验方式
在特殊的环境中如C和SV交互的环境，我们尝尝用不到VIP内置的monitor和scoreboard去校验数据，所以需要自己连接一个monitor去校验数据。

## 一、断开原生的monitor连接
一般在XXX_basic_env.sv中的connect_phase进行monitor的连接，将其注释即可断开连接。

## 二、连接自定义的monitor（RX)
一般我们选择在XXX_base_test.sv中进行自定义monitor的连接。以下是demo：
```systemverilog
`uvm_analysis_imp_decl(_tr_master_rx)  //defien imp

class XXX_base_test extends XXX_test;
    //declare imp
    uvm_analysis_imp_tr_master_rx #(XXX_transaction, XXX_base_test) imp_collected_master_rx;
    XXX_transaction master_rx_trans[$];

    virtual function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        //init imp
        imp_collected_master_rx = new("imp_collected_master_rx", this);
    endfunction : build_phase

    function void connect_phase(uvm_phase phase);
        super.connect_phase(phase);
        XXX_env.XXX.connect(imp_collected_master_rx);//第一步注释的地方移到此处即可
    endfunction : connect_phase

    //define collect function
    function void write_tr_master_rx(XXX_transaction tr);
        master_rx_trans.push_back(tr);
        //print the collected data
        foreach (master_rx_trans[i]) begin
            `uvm_info("master_rx_trans", $sformatf("master_rx_trans[%d]: 0x%08h", i, master_rx_trans[i].data[0]), UVM_NONE);
        end
    endfunction : write_tr_master_rx
endclass
```
在校验数据的时候即可对收集到的数据与C中发送的激励进行校验。

## 三、自定义sequence激励
在case的sv中定义如下随机数：
```systemverilog
class tXXXXXX extends XXX_test;
    uvm_queue#(bit [7:0]) list = new();
    bit [7:0] rand_data_list[$];

    virtual function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        //init the random list
        if(!std::randomize(rand_data_list) with { rand_data_list.size() == 8; }) begin
            `uvm_error(get_full_name(), "fail to randomize a dynamic array")
        end

        foreach(rand_data_list[i]) begin
            list.push_back(rand_data_list[i]);
        end

        //configdb set the list to sequence
        uvm_config_db#(uvm_queue#(bit[7:0]))::set(this, "sequence的full_path", "rand_data_list", list);

    endfunction : build_phase
endclass : tXXXXXX
```
接下来在sequence中添加config_db去获取随机数
```systemverilog
class XXX_sequence extends uvm_sequence #(XXX_transaction);
    virtual task body();
        uvm_queue#(bit[7:0]) rand_data_list;

        if(!uvm_config_db#(uvm_queue#(bit [7:0]))::get(null, get_full_name(), "rand_data_list", rand_data_list)) begin
            `uvm_error(get_full_name(), "fail to get the random data list form top")
        end
        //随后在uvm_do_with初始化激励的时候可以使用rand_data_list.get(i)来取得list中的数据
    endtask
endclass
```