# UVM-phase 机制
通过phase机制可以清晰地将UVM仿真阶段层次化。这里的层次化，不只是各个phase的先后执行顺序，处于同一phase中的层次化组件之间的phase也有先后关系。

## 1、phase执行机制
如果这哪是抛开phase的机制剖析，UVM组件的开发者主要关系各个phase执行的先后顺序。在定义了各个phase虚方法后，UVM环境会按照phase的顺序分别调用这些方法，下图为UVM的phase列表。
|phase|函数/任务|执行顺序|功能|典型应用|
|---|---|---|---|---|
|build|函数|自顶向下|创建和配置测试平台的结构|创建组件和寄存器模型，设置或获取配置|
|connect|函数|自底向上|建立组件之间的连接|连接TLM/TLM2的端口，连接寄存器模型和adapter|
|end_of_elaboration|函数|自底向上|测试环境的微调|显示环境结构，打开文件，为组件添加额外配置|
|start_of_simulation|函数|自底向上|准备测试环境的仿真|显示环境结构，设置断点，设置初始运行的配置值|
|run|任务|自底向上|激励设计|提供激励、采集数据和数据比较，与OVM兼容|
|extract|函数|自底向上|从测试环境中收集数据|从测试平台提取剩余数据，从设计观察最终状态|
|check|函数|自底向上|检查任何不期望的行为|检查不期望的数据|
|report|函数|自底向上|报告测试结果|报告测试结果，将结果写入到文件中|
|final|函数|自顶向下|完成测试活动结束仿真|关闭文件，结束联合仿真引擎|

## 2、run_phase的细分
用户发送激励的一种简单方式是在run_phase中完成上面所有的激励；另一种方式是，如果用户可以将上面几种典型序列划分到不同区间，让对应的激励按区间顺序发送，则可以让测试更有层次。因此，run_phase又分为12个phase:
- pre_reset_phase
- reset_phase
- post_reset_phase
- pre_configure_phase
- configure_phase
- post_configure_phase
- pre_main_phase
- main_phase
- post_main_phase
- pre_shutdown_phase
- shutdown_phase
- post_shutdown_phase
实际上run_phase任务和上面细分的12个phase是并行的，即在start_of_simulation_phase任务执行以后，run_phase和reset_phase开始执行，而在shutdown_phase执行完成之后，需要等待run_phase执行完才可以进入extract_phase。它们的执行顺序关系可以参考下图：

![Phase机制执行顺序](./image/Phase%E6%9C%BA%E5%88%B6%E6%89%A7%E8%A1%8C%E9%A1%BA%E5%BA%8F.png "Phase机制执行顺序")
