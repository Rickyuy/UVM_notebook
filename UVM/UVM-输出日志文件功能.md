# UVM输出日志文件的功能

默认情况下，UVM会将UVM_INFO等信息显示在标准输出(终端屏幕)上。各个仿真器提供将显示在标准输出的信息同时输出到一个日志文件中的功能。但是这个日志文件混杂了所有的UVM_INFO、UVM_WARNING、UVM_ERROR及UVM_FATAL。UVM提供将特定信息输出到特定日志文件的功能，具体的代码如下所示：
```
文件：src/ch3/section3.4/3.4.6/severity/base_test.sv
16 UVM_FILE info_log;
17 UVM_FILE warning_log;
18 UVM_FILE error_log;
19 UVM_FILE fatal_log;
20 virtual function void connect_phase(uvm_phase phase);
21 info_log = $fopen("info.log", "w");
22 warning_log = $fopen("warning.log", "w");
23 error_log = $fopen("error.log", "w");
24 fatal_log = $fopen("fatal.log", "w");
25 env.i_agt.drv.set_report_severity_file(UVM_INFO, info_log);
26 env.i_agt.drv.set_report_severity_file(UVM_WARNING, warning_log);
27 env.i_agt.drv.set_report_severity_file(UVM_ERROR, error_log);
28 env.i_agt.drv.set_report_severity_file(UVM_FATAL, fatal_log);
29 env.i_agt.drv.set_report_severity_action(UVM_INFO, UVM_DISPLAY| UVM_LOG);
30 env.i_agt.drv.set_report_severity_action(UVM_WARNING, UVM_DISPLAY|UVM_LOG);
31 env.i_agt.drv.set_report_severity_action(UVM_ERROR, UVM_DISPLAY| UVM_COUNT | UVM_LOG);
32 env.i_agt.drv.set_report_severity_action(UVM_FATAL, UVM_DISPLAY|UVM_EXIT | UVM_LOG);
…
42 endfunction
```
