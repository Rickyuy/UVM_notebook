# 覆盖实例
```Verilog
module factory_override;
    import uvm_pkg::*;
    'include "uvm_macros.svh"
    class comp1 extends uvm_component;
        'uvm_component_utils(compl)
        function new(string name="comp1", uvm_component parent=null);
            super.new(name, parent);
            $display($sformatf("comp1:: %s is created", name));
        endfunction: new
        virtual function void hello(string name);
            $display($sformatf("comp1:: %s is said hello!", name));
        endfunction
    endclass

    class comp2 extends comp1;
        'uvm_component_utils(comp2)
        function new(string name="comp2", uvm_component parent=null);
            super.new(name, parent);
            $display($sformatf("comp2:: %s is created", name));
        endfunction: new
        virtual function void hello(string name);
            $display($sformatf("comp2:: %s is said hello!", name));
        endfunction
    endclass

    comp1 c1, c2;
    initial begin
        comp1::type_id::set_type_override(comp2::get_type());
        c1 = new("c1");
        c2 = comp1::type_id::create("c2", null);    
        c1.hello("c1");
        c2.hello("c2");
    end
endmodule
```

**输出结果为**
```makefile
comp1:: c1 is created
comp1:: c2 is created
comp2:: c2 is created
comp1:: c1 said hello!
comp2:: c2 said hello!
```