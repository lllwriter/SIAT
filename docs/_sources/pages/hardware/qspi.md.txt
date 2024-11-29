# 2.6 QSPI    
## 2.6.1 QSPI Technical Specification  
![alt text](photo/image-23.png)
###  Description  
SPI_1 is mounted on the MEISHAV100 as an SPI master, responsible for controlling external SPI data transfers. SPI_1 utilizes the open-source axi-spi-master code, which can be referenced at [https://github.com/pulp-platform/axi_spi_master](https://github.com/pulp-platform/axi_spi_master).  
###  Features  
+ Primarily designed for serial NOR Flash and ADC devices
+ Supports standard SPI, dual SPI, or quad-channel SPI commands
   + Signal SD[0] may also be recognized by other   
   + SPI masters as “MOSI,” while SD[3] is commonly referred to as “MISO”
+ Separate FIFOs for RX and TX data
   + 288-byte capacity for TX data and 256-byte capacity for RX data
   + FIFO loading/unloading via 32-bit TL-UL registers
   + Supports arbitrary byte counts in each transaction
+ SPI clock rate controlled by a separate input clock to the core
   + SPI SCK line typically toggles at half the core clock frequency
   + An additional clock rate divider is available to reduce the frequency if needed
+ Supports all SPI polarities and phases (CPOL, CPHA)
   + Additional support for “full-cycle” SPI transactions, where data can be read after an entire SPI clock cycle (as opposed to the typical half cycle in SPI interfaces)
+ Single Transfer Rate (STR) only (i.e., receiving data on multiple lines but only sampling data on one clock edge)
   + Does not support Dual Transfer Rate (DTR)
+ Pass-through mode for coordination with the SPI_DEVICE IP
+ Automatic control of the chip select signal line
+ Compact interrupt footprint: two lines corresponding to two distinct interrupt classes: “error” and “spi_event”
   + Fine-grained interrupt masking provided by an auxiliary enable register     
## 2.6.2 Theory of Operation  
## 2.6.3 QSPI Design Verification  


Connect `spi_1` to the QSPI on MEISHAV100, and use data sent via `spi_1` to control QSPI for reading and writing memory.

### 2.6.3.1 Connecting SPI_1 and QSPI

Add the following code to the top-level file **dut_MEISHAV100_TOP_wrapper.sv**:


```verilog
// spi_1 test

assign spi_slave_if.spi_sdo         =   spi_master_sdo;
assign spi_master_sdi               =   spi_slave_if.spi_sdi;
assign spi_slave_if.spi_clk         =   spi_master_clk;
assign spi_slave_if.spi_csn         =   spi_master_csn[0];

```
`spi_slave_if` is a virtual interface added for testing QSPI, used to connect to QSPI. The `spi_master` signal is the `spi_1` signal brought out at the top level.

### 2.6.3.2 Writing the Test Case

Follow the structure of other test cases, primarily modifying the content within the `main_phase`:


```verilog
tl_agent_pkg::tl_host_single_seq seq;
seq = new("seq");
seq.control_addr_alignment = 1;
seq.control_rand_source = 1;
seq.control_rand_size = 1;
seq.override_a_source_val = 1;
seq.overridden_a_source_val = 2;

`DV_CHECK_RANDOMIZE_WITH_FATAL(seq,
  size             == 2;
  mask             == 'hf;
  source == 2;
  write == 1;
)

// Initialize Mode and clock and data length, command and address length, Dummy cycles

// reset FIFO
seq.addr = `SPI1_REG64_STATUS;
seq.data = ('h1 << 4);
seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));


// set clock, spi_clk = axi_clk / (2 * (clk_div + 1))
seq.addr = `SPI1_REG64_CLKDIV;
seq.data = 32'h1;
seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

// set data length and command to be 8 bits, address 0 bits
seq.addr = `SPI1_REG64_SPILEN;
seq.data = {16'h8, 2'h0, 6'h0, 2'h0, 6'h8};
seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));


// set write dummy cycles to be 0, read dummy cycles to be 33
seq.addr = `SPI1_REG64_SPIDUM;
seq.data = {16'h0, 16'h21};
seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

```

Select the values for the configuration registers based on whether QSPI transfer is used or not.


```verilog
// if transmit by qspi mode, need to set reg0 of qspi to 1 
if (qspi_mode == 1) begin
    seq.addr = `SPI1_REG64_SPICMD;
    seq.data = 8'h01 << 24;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    seq.addr = `SPI1_REG64_STATUS;
    seq.data = 32'h0102;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    seq.addr = `SPI1_REG64_TXFIFO;
    seq.data = 8'h01 << 24;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
end

```

Start sending data from a specific address, and increment the address by 4 after each data transmission.


```verilog
// set data length and address to be 32 bits, command 8 bits
seq.addr = `SPI1_REG64_SPILEN;
seq.data = {16'h20, 2'h0, 6'h20, 2'h0, 6'h8};
seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

// set initial address and data
rw_addr = 32'h5003_F060;
write_data = 32'h1;

repeat(2000) begin
    `uvm_info(`gfn, $sformatf("spi_1 send data: %h", write_data), UVM_NONE)
    // write data to rw_addr
    seq.write = 1;
    seq.addr = `SPI1_REG64_SPICMD;
    seq.data = 32'h0200_0000;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    seq.addr = `SPI1_REG64_SPIADR;
    seq.data = rw_addr;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    if (qspi_mode == 1) begin
        seq.addr = `SPI1_REG64_STATUS;
        seq.data = 32'h0108;
        seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
    end
    else begin
        seq.addr = `SPI1_REG64_STATUS;
        seq.data = 32'h0102;
        seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
    end

    seq.addr = `SPI1_REG64_TXFIFO;
    seq.data = write_data;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
    #1000ns;

    // read data from rw_addr, data will be stored in rxfifo
    seq.addr = `SPI1_REG64_SPICMD;
    seq.data = 32'h0b00_0000;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    seq.addr = `SPI1_REG64_SPIADR;
    seq.data = rw_addr;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    if (qspi_mode == 1) begin
        seq.addr = `SPI1_REG64_STATUS;
        seq.data = 32'h0104;
        seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
    end
    else begin
        seq.addr = `SPI1_REG64_STATUS;
        seq.data = 32'h0101;
        seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));
    end
    #1000ns;

    // read data from spi_1 rxfifo
    seq.addr = `SPI1_REG64_RXFIFO;
    seq.write = 0;
    seq.start(env.virtual_sequencer.match_sqr("tl", "tile-RocketTile*tl_mem_mst"));

    rw_addr += 32'h4;
    write_data += 32'h1;
end

```

### 2.6.3.3 Result of Test

Primarily test the address spaces for DDR and IRAM. The testing method involves separately testing the low, middle, and high parts of both address spaces. A portion of addresses in each part are tested. The test results are as follows:

Address | Result | Issue
:-----------: | :-----------: | :-----------:
0x8000_0000~0x8000_9C40 (10000 addresses) | Correct | 
0x9FFF_D1C0~0xA000_130C (10000 addresses) | Correct | 
0xBFFF_63C0~0xC000_0000 (10000 addresses) | Partial Errors | The last three addresses read data as 0. After comparing waveforms and code, it was found to be an issue with the mounted VIP. Setting the working mode of the VIP mounted to `spi_1` to SLAVE resulted in correct operation.
0x5000_0000~0x5000_1F40 (2000 addresses) | Correct | 
0x5003_F060~0x5004_0FA0 (2000 addresses) | Correct | 
0x5007_E0C0~0x5008_0000 (2000 addresses) | Correct | 

### 2.6.3.4 Software Testing

Translate the above SystemVerilog test code into C language code for software testing. The C code is as follows:



```c
// See LICENSE for license details.
#include <stdint.h>


#include <platform.h>

#include "common.h"
#include "spi1.h"

#define QSPI 1

#define DEBUG
#include "kprintf.h"

int main(void)
{
    long long addr[] = {0x80000000, 0x9FFFF830, 0xBFFFF060, 0x50000000, 0x50003830, 0x50007060};
  
    REG32(uart, UART_REG_TXCTRL) = UART_TXEN;  // donot delete

    kprintf("Test begin ! @ core: %x", read_csr(mhartid));

    // reset fifo
    _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = (1 << 4);

    // set clock, spi_clk = axi_clk / (2 * (clk_div + 1))
    _REG32(SPI1_BASE_ADDR, SPI1_REG_CLKDIV) = 0x01;

    // set data length and command to be 8 bits, address 0 bits
    _REG32(SPI1_BASE_ADDR, SPI1_REG_SPILEN) = (0x08 << 16) | 0x08;
    _REG32(SPI1_BASE_ADDR, SPI1_REG_SPIDUM) = 0x21;

    // enable qspi transmit
    if (QSPI == 1) {
        _REG32(SPI1_BASE_ADDR, SPI1_REG_SPICMD) = (0x01 << 24);
        _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = 0x0102;
        _REG32(SPI1_BASE_ADDR, SPI1_REG_TXFIFO) = (0x01 << 24);
    }

    // set data length and address to be 32 bits, command 8 bits
    _REG32(SPI1_BASE_ADDR, SPI1_REG_SPILEN) = (0x20 << 16 | 0x20 << 8 | 0x08);

    for (int i = 0; i < 6; ++ i) {
        // set initial address and data
        int data = 0x12345678;
        long long rw_addr = addr[i];
        if (i == 0) kprintf("DDR test begin!");
        else if (i == 3) kprintf("IRAM test begin!");
        for (int j = 1; j <= 1000; ++ j) {
            // wait until SPI is idle
            while ((_REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) & 0x01) == 0);

            // set write command and address
            _REG32(SPI1_BASE_ADDR, SPI1_REG_SPICMD) = (0x02 << 24);
            _REG32(SPI1_BASE_ADDR, SPI1_REG_SPIADR) = rw_addr;

            // initiate a write operation with select CS0
            if (QSPI == 1) _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = 0x0108;
            else _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = 0x0102;

            // wait until tx buffer has available place
            while (((_REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) >> 24) & 0xFF) >= 8);

            // write data to addr
            _REG32(SPI1_BASE_ADDR, SPI1_REG_TXFIFO) = data;

            // wait until SPI is idle
            while ((_REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) & 0x01) == 0);

            // set read command and address
            _REG32(SPI1_BASE_ADDR, SPI1_REG_SPICMD) = (0x0b << 24);
            _REG32(SPI1_BASE_ADDR, SPI1_REG_SPIADR) = rw_addr;

            // initiate a write operation with select CS0
            if (QSPI == 1) _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = 0x0104;
            _REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) = 0x0101;

            // wait until rx buffer has available place
            while (((_REG32(SPI1_BASE_ADDR, SPI1_REG_STATUS) >> 16) & 0xFF) == 0);

            // read data from rxfifo
            int read_data = _REG32(SPI1_BASE_ADDR, SPI1_REG_RXFIFO);

            if (read_data != data) {
                kprintf("SPI1_TEST FAIL! at addr: %x, the correct data is: %x, but get: %x", rw_addr, data, read_data);
            }
            else kprintf("SPI1_TEST PASS! at addr: %x", rw_addr);

            rw_addr += 0x4;
            data += 0x1;
        }
        if (i == 2) kprintf("DDR test end!");
        else if (i == 5) kprintf("IRAM test end!");
    }
    
    kprintf("Test end ! @ core: %x", read_csr(mhartid));


    if (1){
        kprintf("PASS");
    }
    else{
        kprintf("FAIL");
    }

    kprintf(SHUTDOWN_FLAG_STR);  // donnot delete

    return 0;
}


```

Run the test case `soft_base_test.sv`. Mount the AXI VIP in the `initialize` function of the `soft_base_test_cfg` class to ensure QSPI operates normally.


```verilog
foreach(this.axi_agent_sw[i]) begin
    this.axi_agent_sw[i] = dv_utils_pkg::ON;
    if(uvm_is_match("*vc707mig1gb*s_axi_slv*", i)) begin
        this.axi_agent_work_mode[i] = dv_utils_pkg::SLAVE;
    end
    else if(uvm_is_match("*u_axi_spi_slave*m_axi_mst*", i)) begin // for qspi
        this.axi_agent_work_mode[i] = dv_utils_pkg::SLAVE;
    end
    else if(uvm_is_match("*vc707axi_to_pcie_x1*m_axi_slv*", i)) begin
        this.axi_agent_work_mode[i] = dv_utils_pkg::ONLY_MONITOR;
    end
    else if(uvm_is_match("*vc707axi_to_pcie_x1*s_axi_mst*", i)) begin
        this.axi_agent_work_mode[i] = dv_utils_pkg::SLAVE;
    end
    else begin
        this.axi_agent_work_mode[i] = dv_utils_pkg::ONLY_MONITOR;
    end
    `uvm_info(get_type_name(), $sformatf("set_vip_agent_work_mode for inst_path %s, work_mode %s", i, this.axi_agent_work_mode[i].name()), UVM_DEBUG);
    //this.m_axi_agent_cfg[i].if_mode = dv_utils_pkg::Host;  // donnot need add in here
end

```

Then, run this test case, compile the above C code into a hex file and load it into the RAM. Run the simulation, and the simulation results are as follows:

Address | Result | Issue
:-----------: | :-----------: | :-----------:
0x8000_0000~0x8000_0FA0 (500 addresses) | Correct | 
0x9FFF_F830~0xA000_07D0 (500 addresses) | Correct | 
0xBFFF_F060~0xC000_0000 (500 addresses) | Correct | 
0x5000_0000~0x5000_0FA0 (500 addresses) | Correct | 
0x5003_F830~0x5004_07D0 (500 addresses) | Correct | 
0x5007_F060~0x5008_0000 (500 addresses) | Correct | 




## 2.6.4 Hardware Interfaces    

Signal | Direction | Description
:-----------: | :-----------: | :-----------:
spi_clk | output | Master Clock
spi_csn0 | output | Chip Selcet0
spi_csn1 | output | Chip Selcet1
spi_csn2 | output | Chip Selcet2
spi_csn3 | output | Chip Selcet3
spi_mode[1:0] | output | SPI Mode
spi_sdo0 | output | Output Line0
spi_sdo1 | output | Output Line1
spi_sdo2 | output | Output Line2
spi_sdo3 | output | Output Line3
spi_sdi0 | input | Input Line0
spi_sdi1 | input | Input Line1
spi_sdi2 | input | Input Line2
spi_sdi3 | input | Input Line3    

## 2.6.5 Registers  

According to the pulpino documentation, the control registers and related functionalities for SPI_1 are as follows:  

Offset | Name
:-----------:| :-----------:
0x00 | STATUS
0x08 | CLKDIV
0x10 | SPICMD
0x18 | SPIADR
0x20 | SPILEN
0x28 | SPIDUM
0x40 | TXFIFO
0x80 | RXFIFO


### STATUS（Status Register）  

**Reset Value：0x0000_0000**
![alt text](photo/38481e84-dde1-44ed-b3ca-65937ab976b5.png)
- **Bit 11: 8  CS: Chip Select**
Designates the chip select signal to be used for the next transfer.

- **Bit 4   SRST: Software Reset**
Clears the FIFO and terminates the ongoing transfer.

- **Bit 3   QWR: Quad Write Command**
Executes a write operation in QuadSPI mode.

- **Bit 2   QRD: Quad Read Command**
Executes a read operation in QuadSPI mode.

- **Bit 1   WR: Write Command**
Executes a write operation in standard SPI mode.

- **Bit 0   RD: Read Command**
Executes a read operation in standard SPI mode.

### CLKDIV（Clock Divider）

**Reset Value：0x0000_0000**
![alt text](photo/5327509c-1857-4e83-8258-c9a460d65269.png)
- **Bit 7:0  CLKDIV: Clock Divider**
The clock divider value, used to divide the SoC clock for SPI transfers. This register should not be modified during a transfer.

### SPICMD（SPI Command）

**Reset Value：0x0000_0000**  

![alt text](photo/be818689-11e9-4643-9b37-3969e3aa2e00.png)

- **Bit 31:0  SPICMD: SPI Command**
When executing a read or write transfer, the SPI command is sent first before any data is read or written. The length of the SPI command can be controlled using the SPILEN register.

### SPIADR（SPI Address）

**Reset Value：0x0000_0000**

![alt text](photo/3f6ab307-a8e4-4258-a1c4-b2b35847373c.png)

- **Bit 31:0  SPIADR: SPI Address**
When performing a read or write transfer, the SPI command is sent first, followed by the SPI address, before any data is read or written. The length of the SPI address can be controlled using the SPILEN register.

###  SPILEN（SPI Transfer Length）

**Reset Value：0x0000_0000**

![alt text](photo/bf22607e-923e-4f6f-8679-bad952a7b315.png)

- **Bit 31:16  DATALEN: SPI Data Length**
The number of bits to be read or written. Note that the SPI command and address are first written to the SPI slave device.

- **Bit 13:8  ADDRLEN: SPI Address Length**
The number of bits for the SPI address that should be sent.

- **Bit 5:0  SPI Command Length**
The number of bits for the SPI command that should be sent.  

#### SPIDUM（SPI Dummy Cycles）

**Reset Value：0x0000_0000**

![alt text](photo/fead8b77-1e4c-495d-947c-26f0f200ff17.png)
  
- **Bit 31:16  DUMMYWR: Write Dummy Cycles**
The number of dummy cycles (no write or read) between sending the SPI command + SPI address and writing data.

- **Bit 15:0  DUMMYRD: Read Dummy Cycles**
The number of dummy cycles (no write or read) between sending the SPI command + SPI address and reading data.  


### TXFIFO (SPI Transmit FIFO)

**Reset Value: 0x0000_0000**

![alt text](photo/273da941-0aed-463d-b909-8f4df84aa9bb.png)

- **Bit 31:0  TX: Transmit Data**
Data to be written into the FIFO for transmission.
  

### RXFIFO (SPI Receive FIFO)

**Reset Value: 0x0000_0000**

![alt text](photo/3b08a5ca-6b21-4ab6-8dee-8c326cb5e1f5.png)

- **Bit 31:0  RX: Receive Data**
Data to be read from the FIFO received during transmission.

Note: There seems to be a typo in the original text where "Transmit Data" is mentioned for both TX and RX. In the RXFIFO section, it should be "Receive Data" as corrected in the translation.   
## 2.6.6 Checklist