# UVM的知识点
## 1、phase的执行顺序
build phase的执行顺序是从树根到树叶——先执行env的build_phase，再执行agent的build_phase，最后执行driver和monitor的build_phase。

connect_phase的执行顺序是从树叶到树根——先执行driver和monitor的connect_phase，再执行agent的connect_phase，最后执行env的connect_phase。

## 2、uvm_component与uvm_object的区别
uvm_object是UVM中最基本的类，读者能想到的几乎所有的类都继承自uvm_object，包括uvm_component。uvm_component派生自uvm_object，说明uvm_component拥有uvm_object的特性，同时又有自己的一些特质。但是uvm_component的一些特性，uvm_object则不一定具有。这是面向对象编程中经常用到的一条规律。

uvm_component有两大特性是uvm_object所没有的，一是通过在new的时候指定parent参数来形成一种树形的组织结构，二是有phase的自动执行特点。

UVM常用类的继承关系如下图所示：

![UVM常用类的继承关系](./image/uvm%E5%B8%B8%E7%94%A8%E7%B1%BB%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png "UVM常用类的继承关系")

从图中可以看出，从uvm_object派生出了两个分支，所有的UVM树的结点都是由uvm_component组成的，只有基于uvm_component派生的类才可能成为UVM树的结点；最左边分支的类或者直接派生自uvm_object的类，是不可能以结点的形式出现在UVM树上。

那么有哪些类会派生自uvm_object呢？

答案是除了派生自uvm_component类之外的类，几乎所有的类都派生自uvm_object。换个说法，除了driver、monitor、agent、model、scoreboard、env、test之外几乎所有的类，本质上都是uvm_object，如sequence、sequence_item、transaction、config等。

>**一般来说有生命周期的的类都派生自uvm_object，没有生命周期在整个仿真期间一直存在的类派生自uvm_component。**

