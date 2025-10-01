

---

# **APB-to-I2C Bridge UVM Testbench Repository**

---

## **1. APB-to-I2C Bridge Overview**

### **1.1 Specification**

The **APB-to-I2C Bridge** connects an **APB master** to an **I2C slave**, enabling communication between APB-based SoCs and I2C peripherals.

**Key Features:**

* Supports **APB write, read, and write-read transactions**.
* Supports **I2C write and read transactions**.
* Configurable **APB address/data width**.
* Handles **I2C START, STOP, and ACK/NACK** conditions.

---

### **1.2 APB Protocol Basics**

APB signals:

* **PADDR**: Address bus
* **PSEL**: Peripheral select
* **PENABLE**: Enables transfer
* **PWDATA/PRDATA**: Data bus
* **PWRITE**: Read/write control
* **PREADY**: Peripheral ready
* **PSLVERR**: Error signal

---

### **1.3 I2C Protocol Basics**

I2C is a **two-wire serial protocol**:

* **SDA**: Serial Data
* **SCL**: Serial Clock
* Master initiates communication with **START/STOP**.
* Supports **ACK/NACK** handshaking.

---

## **2. APB-to-I2C Bridge Interface**

**File:** `src/apb_i2c_if.sv`

```systemverilog
interface apb_i2c_if #(parameter ADDR_WIDTH=8, DATA_WIDTH=8)
                     (input logic pclk, presetn, scl_in, sda_in);
  
  // APB Signals
  logic [ADDR_WIDTH-1:0] paddr;
  logic [DATA_WIDTH-1:0] pwdata;
  logic [DATA_WIDTH-1:0] prdata;
  logic                  pwrite;
  logic                  psel;
  logic                  penable;
  logic                  pready;
  logic                  pslverr;

  // I2C Signals
  logic scl;
  logic sda;
  
endinterface
```

* Combines APB and I2C signals.
* Used by **drivers, monitors, and sequences**.

---

## **3. Transaction Classes**

**File:** `src/apb_i2c_pkg.sv`

```systemverilog
package apb_i2c_pkg;
  import uvm_pkg::*;
  `include "uvm_macros.svh"

  typedef enum {WRITE, READ, WRITE_READ} apb_trans_e;
  typedef enum {I2C_WRITE, I2C_READ} i2c_trans_e;

  class apb_trans extends uvm_sequence_item;
    rand logic [7:0] addr;
    rand logic [7:0] data;
    rand apb_trans_e type;
    `uvm_object_utils(apb_trans)
    function new(string name="apb_trans");
      super.new(name);
    endfunction
  endclass

  class i2c_trans extends uvm_sequence_item;
    rand logic [7:0] data;
    rand logic [6:0] addr;
    rand i2c_trans_e type;
    `uvm_object_utils(i2c_trans)
    function new(string name="i2c_trans");
      super.new(name);
    endfunction
  endclass

endpackage
```

* **apb_trans**: write, read, write-read
* **i2c_trans**: write, read
* `rand` enables **constrained-random UVM testing**

---

## **4. APB Driver**

**File:** `src/apb_driver.sv`

```systemverilog
class apb_driver extends uvm_driver #(apb_trans);
  `uvm_component_utils(apb_driver)
  virtual apb_i2c_if vif;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      apb_trans tr;
      seq_item_port.get_next_item(tr);
      
      // APB write
      if(tr.type == WRITE) begin
        vif.paddr   <= tr.addr;
        vif.pwdata  <= tr.data;
        vif.pwrite  <= 1;
        vif.psel    <= 1;
        vif.penable <= 1;
        @(posedge vif.pclk);
        wait(vif.pready);
        vif.psel    <= 0;
        vif.penable <= 0;
      end
      // APB read
      else if(tr.type == READ) begin
        vif.paddr   <= tr.addr;
        vif.pwrite  <= 0;
        vif.psel    <= 1;
        vif.penable <= 1;
        @(posedge vif.pclk);
        wait(vif.pready);
        tr.data = vif.prdata;
        vif.psel    <= 0;
        vif.penable <= 0;
      end
      // APB write-read
      else if(tr.type == WRITE_READ) begin
        // First write
        vif.paddr   <= tr.addr;
        vif.pwdata  <= tr.data;
        vif.pwrite  <= 1;
        vif.psel    <= 1;
        vif.penable <= 1;
        @(posedge vif.pclk);
        wait(vif.pready);
        vif.psel    <= 0;
        vif.penable <= 0;
        // Then read
        vif.paddr   <= tr.addr;
        vif.pwrite  <= 0;
        vif.psel    <= 1;
        vif.penable <= 1;
        @(posedge vif.pclk);
        wait(vif.pready);
        tr.data = vif.prdata;
        vif.psel    <= 0;
        vif.penable <= 0;
      end
      seq_item_port.item_done();
    end
  endtask
endclass
```

---

## **5. I2C Driver**

**File:** `src/i2c_driver.sv`

```systemverilog
class i2c_driver extends uvm_driver #(i2c_trans);
  `uvm_component_utils(i2c_driver)
  virtual apb_i2c_if vif;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      i2c_trans tr;
      seq_item_port.get_next_item(tr);

      // Generate I2C transaction
      if(tr.type == I2C_WRITE) begin
        // START
        vif.sda <= 0; @(posedge vif.scl);
        // Send address + write bit, then data bytes
        // Generate STOP
        vif.sda <= 1; @(posedge vif.scl);
      end
      else if(tr.type == I2C_READ) begin
        // START
        vif.sda <= 0; @(posedge vif.scl);
        // Send address + read bit
        // Read data
        // Generate STOP
        vif.sda <= 1; @(posedge vif.scl);
      end
      seq_item_port.item_done();
    end
  endtask
endclass
```

---

## **6. APB Monitor**

**File:** `src/apb_monitor.sv`

```systemverilog
class apb_monitor extends uvm_monitor;
  `uvm_component_utils(apb_monitor)
  virtual apb_i2c_if vif;
  uvm_analysis_port #(apb_trans) ap;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      if(vif.psel && vif.penable) begin
        apb_trans tr = apb_trans::type_id::create("tr");
        tr.addr = vif.paddr;
        tr.data = vif.prdata;
        tr.type = vif.pwrite ? WRITE : READ;
        ap.write(tr);
      end
      @(posedge vif.pclk);
    end
  endtask
endclass
```

---

## **7. I2C Monitor**

**File:** `src/i2c_monitor.sv`

```systemverilog
class i2c_monitor extends uvm_monitor;
  `uvm_component_utils(i2c_monitor)
  virtual apb_i2c_if vif;
  uvm_analysis_port #(i2c_trans) ap;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      @(posedge vif.scl);
      i2c_trans tr = i2c_trans::type_id::create("tr");
      ap.write(tr);
    end
  endtask
endclass
```

---

## **8. Scoreboard**

**File:** `src/scoreboard.sv`

```systemverilog
class scoreboard extends uvm_component;
  `uvm_component_utils(scoreboard)
  uvm_analysis_imp #(apb_trans, scoreboard) apb_imp;
  uvm_analysis_imp #(i2c_trans, scoreboard) i2c_imp;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    apb_imp = new("apb_imp", this);
    i2c_imp = new("i2c_imp", this);
  endfunction

  function void write_apb(apb_trans tr);
    $display("APB Transaction: addr=%0h data=%0h type=%0s", tr.addr, tr.data, tr.type.name());
  endfunction

  function void write_i2c(i2c_trans tr);
    $display("I2C Transaction: addr=%0h data=%0h type=%0s", tr.addr, tr.data, tr.type.name());
  endfunction
endclass
```

---

## **9. Environment**

**File:** `src/env.sv`

```systemverilog
class env extends uvm_env;
  `uvm_component_utils(env)

  apb_driver   apb_drv;
  i2c_driver   i2c_drv;
  apb_monitor  apb_mon;
  i2c_monitor  i2c_mon;
  scoreboard   sb;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task build_phase(uvm_phase phase);
    apb_drv = apb_driver::type_id::create("apb_drv", this);
    i2c_drv = i2c_driver::type_id::create("i2c_drv", this);
    apb_mon = apb_monitor::type_id::create("apb_mon", this);
    i2c_mon = i2c_monitor::type_id::create("i2c_mon", this);
    sb      = scoreboard::type_id::create("sb", this);
  endtask

  task connect_phase(uvm_phase phase);
    apb_mon.ap.connect(sb.apb_imp);
    i2c_mon.ap.connect(sb.i2c_imp);
  endtask
endclass
```

---

## **10. Test Sequences**

**File:** `src/test_sequences.sv`

```systemverilog
class apb_write_seq extends uvm_sequence #(apb_trans);
  `uvm_object_utils(apb_write_seq)

  task body();
    apb_trans tr;
    tr = apb_trans::type_id::create("tr");
    tr.addr = 8'h10; tr.data = 8'hAA; tr.type = WRITE;
    start_item(tr);
    finish_item(tr);
  endtask
endclass

class apb_read_seq extends uvm_sequence #(apb_trans);
  `uvm_object_utils(apb_read_seq)

  task body();
    apb_trans tr;
    tr = apb_trans::type_id::create("tr");
    tr.addr = 8'h10; tr.type = READ;
    start_item(tr);
    finish_item(tr);
  endtask
endclass
```

---

## **11. Top-Level Testbench**

**File:** `tb/apb_i2c_tb.sv`

```systemverilog
module apb_i2c_tb;
  logic pclk, presetn, scl, sda;
  apb_i2c_if vif(pclk, presetn, scl, sda);

  env uvm_env_inst;

  initial begin
    pclk = 0;
    forever #5 pclk = ~pclk;
  end

  initial begin
    presetn = 0;
    #20 presetn = 1;
  end

  initial begin
    uvm_env_inst = env::type_id::create("uvm_env_inst");
    run_test();
  end
endmodule
```

---

## **12. Example Transaction Outputs**

### **12.1 APB Write Transaction**

**Description:** Writes data to I2C peripheral via APB.

![APB-I2C Bridge Transaction_page-0001](https://github.com/user-attachments/assets/7d560530-6034-4fbf-945b-69635f537b94)

---

### **12.2 APB Read Transaction**

**Description:** Reads data from I2C peripheral via APB.

![APB-I2C Bridge Transaction_page-0002](https://github.com/user-attachments/assets/0d292f86-f755-4d6d-9243-4a0a40021805)


---

### **12.3 APB Write-Read Transaction**

**Description:** Writes then reads data to/from I2C peripheral.

![APB-I2C Bridge Transaction_page-0003](https://github.com/user-attachments/assets/f83351c3-8ba9-43bf-bd9d-d3ede62631c7)


---

### **12.4 I2C Write Transaction**

**Description:** Direct I2C write operation.

![APB-I2C Bridge Transaction_page-0004](https://github.com/user-attachments/assets/12a89102-ba89-41ce-845b-96ec989289ae)


---

### **12.5 I2C Read Transaction**

**Description:** Direct I2C read operation.

![APB-I2C Bridge Transaction_page-0005](https://github.com/user-attachments/assets/3b981be2-f670-4fd6-b697-b3c573594ad3)


---

## **13. Applications, Advantages, and Limitations**

**Applications:**

* Microcontrollers communicating with **sensors or EEPROMs**
* Embedded systems needing **APB-to-I2C bridging**
* FPGA designs with APB master cores

**Advantages:**

* Seamless APB-to-I2C communication
* Supports multiple transaction types
* UVM-compliant testbench
* Easy to extend for **functional coverage**

**Limitations:**

* Limited to **standard I2C speeds** (fast-mode extension possible)
* 8-bit APB/I2C data width by default
* Timing depends on APB and I2C clock synchronization

---

