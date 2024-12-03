
# 2.4 SPI    
## 2.4.1 SPI Technical Specification    

![alt text](photo/image-21.png)  

###  Description  
The SPI controller supports single-, dual-, and quad-channel protocols in host-only mode. The baseline controller provides a FIFO-based interface for executing programmed input/output operations. Software initiates transfers by queuing frames into the transmit FIFO; upon completion, the slave responses are placed in the receive FIFO.

Additionally, the dedicated SPI0 controller implements an SPI flash read sequencer, which exposes the contents of an external SPI flash as a read-only/execute memory-mapped device. Given an input clock rate below 100 MHz and assuming the external SPI flash device supports the common Winbond/Numonyx serial read (0x03) command, the SPI0 controller resets to a state that allows memory-mapped reads. To enhance performance, consecutive accesses are automatically merged into a single long read command.

The fctrl register governs the toggling between memory-mapped and programmed I/O modes. In programmed I/O mode, memory-mapped reads do not access the external SPI flash device and instead return 0 immediately. Hardware interlocks ensure that the current transfer completes before mode transitions and control register updates take effect.   

## 2.4.2 Design Verification      
 **Goals** 

Verify SPI IP features by running dynamic simulations with a SV/UVM based testbench    

**Design features**    

For detailed information on SPI design features, please see the **SPI Technical Specification**.   

**Testbench architecture**    

Top level testbench is located at /trunk/hw/d2dv100_top/dv/dut_MEISHAV100_TOP_wrapper.sv. It instantiates the TOP DUT module /trunk/hw/d2dv100_top/rtl/top/MEISHAV100_TOP.sv. In addition, it instantiates the following interfaces, connects them to the DUT and sets their handle into `uvm_config_db`:  
+  Clock and reset interface  
+  Tilelink host interface  
+  SPI IOs    
  
**Global types & methods**  

All common types and methods defined at the package level can be found in `spi_env_pkg`    

**TL_agent**  

SPI Device instantiates (already handled in CIP base env) tl_agent which provides the ability to drive and independently monitor random traffic via TL host interface into SPI Device.  

**SPI_agent**  

`spi agent` is used to drive and monitor SPI items.
### 2.4.2.1 Testplan  
This validation environment encompasses a substantial amount of supplementary code; however, the parent classes may introduce anomalies or superfluous functionalities. Consequently, it is permissible to directly alter the inherited parent class to the corresponding UVM base class, as necessitated  
#### 2.4.2.1.1 Develop an agent  

Within this development environment, the SPI operates as a slave_agent driven by the CPU. Consequently, the SPI_agent comprises a single driver responsible for capturing signals emitted by the SPI ports during the testing process.    
##### spi_agent_cfg.sv  

This file defines properties such as the SPI polarity, phase, endianness, and declares an SPI interface for the monitor to collect data. The core function, `wait_sck_edge`, is pivotal as it determines the sampling timing based on the SPI’s polarity and phase.
#####  spi_agent_cov.sv
The coverage-related file is currently empty.
#####  spi_agent_pkg.sv

This file imports the libraries and modules required by the `SPI agent` and also defines various enumerated variables, such as csmode and transfer modes (single, dual, quad), enhancing convenience for usage.

#####  spi_agent.sv

This defines an `spi_monitor` and `spi_config`, where the configured spi_if from config_db is passed to `spi_cfg`.`spi_if`. Additionally, spi_cfg is set in the database.
##### spi_device_driver.sv
The slave_spi driver currently remains unused.

##### spi_driver.sv

The base class for the spi_driver, from which both the master_spi and slave_spi drivers inherit. It defines two virtual tasks: reset and drive. Subclasses can extend it with additional tasks if neede
##### spi_host_driver.sv
The master_spi driver is currently inactive.

#####  spi_if.sv

The SPI interface, designed to supply data to the monitor, encompasses the four fundamental SPI signals along with additional debug signals.

#####  spi_item.sv

The data actually transmitted by the spi_sequence entails a queue for storing 8-bit data and the transfer type (either write or read).
#####  spi_monitor.sv

The primary components of this file are the definition of two `spi_items` and two `uvm_analysis_ports`, which collect SPI transmission data and facilitate communication with the scoreboard. The core task, collect_trans, is responsible for gathering SPI transmission data. Given that the SPI chip select (CS) signal is active low, collection commences upon detecting a low state on CS. The process involves waiting for a rising edge (conditional on clock polarity and phase) before sampling the MISO signal for validity and storing it in a predefined array (considering endianness). Additionally, the spi_monitor incorporates rudimentary validation by initializing an incremental variable, exp, and comparing it to each collected 8-bit segment to verify data accuracy. Finally, the received data is transmitted to the host_analysis_port for extraction by the scoreboard.

##### spi_sckmode_c.sv
It is intended to randomize values within the sckmode, but given the current SPI configuration, it is not feasible to write to the corresponding registers via the bus. Consequently, this file remains inactive for the time being.
##### spi_sequencer.sv
The sequencer, designated for transmitting SPI data, currently remains vacant.

##### spi_base_agent.core
The `.core` file serves as a configuration management tool for projects in FuseSoC, allowing you to replicate existing configurations and modify their components as needed:

- **name**: Set to "lowrisc:dv:XXX:0.1", where XXX should be replaced with the corresponding name for your project. This name must align with any references to this configuration within other files.
- **depend**: Specify dependencies on other configuration files here, such as an environment dependency on an agent.
- **files**: This section lists all files included in the current configuration, encompassing all files from the referenced spi_agent. If file 'a' is included in file 'b', attribute {is_include_file: true} should be appended to file 'a'.

For additional details, consult the official FuseSoC documentation at:

[https://fusesoc.readthedocs.io/en/stable/user/build_system/index.html](https://fusesoc.readthedocs.io/en/stable/user/build_system/index.html).

#### 2.4.2.1.2 Develop an env

##### spi_scoreboard.sv


Four queues are defined to store the expected and actual write and read data. The expected data is sourced from the TL-agent, while the actual data is obtained from the SPI-agent. SPI data is retrieved via uvm_blocking_get_port, and a comparison is made between the expected and actual data. However, there are inherent issues with the current scoreboard, rendering it ineffective in the validation environment used.

#####  spi_env.sv
The components include an spi_agent, an spi_agent_cfg, an spi_scoreboard, and a uvm_tlm_analysis_fifo which serves to connect the spi_scoreboard with the spi_monitor.

#####  spi_env_pkg.sv

The necessary libraries and files for the spi_env are imported, and an enumeration variable, spi_host_intr_e, is defined (currently unused) to facilitate utilization

#####  spi_base_env.core

The configuration file for the spi_env, designed to provide dependencies for the spi_test.

#### 2.4.2.1.3 Develop rals
please translate this passage into more professional English by changing sentence structure,grammactical structure,words,etc

#### 2.4.2.1.4 Develop rtls
To validate SPI read operations, an SPI-slave device was integrated into our SOC, enabling data transmission from the SOC's SPI interface to the SPI-slave. The objective was to verify whether the SPI-slave could accurately relay the correct values back.

The RTL code for this purpose was sourced from a GitHub project (

[https://github.com/SherifMohamed2602/SPI_Interface](https://github.com/SherifMohamed2602/SPI_Interface)

) authored by Sherif Mohamed. The project includes instructions for running tests, which were successfully validated through experimentation. According to the author's implementation, the testing protocol involves sending 11-bit data segments. The first bit alters the SPI's state (e.g., write data, read data, write address). The second and third bits indicate to the SPI-slave's memory to store or transmit corresponding addresses or data. The remaining eight bits constitute the actual data (address or data) being written. This test case methodology was employed to conduct the SPI read validation.


#### 2.4.2.1.5 Develop seq_libs

The base sequence for SPI, which is currently not utilized as the current environment employs the spi_item from the identical agent folder.


#### 2.4.2.1.6 Develop tests

This test is adapted from the pre-existing tl-test within the reference environment.


##### spi_macros.svh
The base address and offset definitions for SPI registers facilitate efficient usage.

##### spi_test.sv
The test was written with reference to the tlul_test.sv, primarily modifying the code within the main_phase. Additional components include the definition of an spi_if within spi_base_test_cfg and an spi_env within spi_base_test.

Within the main_phase, data is sent to SPI registers via the tl-agent to configure them (as an alternative to the ral method). Data is then transmitted to the spi_txfifo, initiating automatic forwarding by the SPI. Accuracy of the mosi port's output is verified through monitoring. Given that the txfifo can only transmit one byte of data at a time, test data ranging from 0x00 to 0xff is utilized to ensure comprehensive coverage.

The aforementioned process constitutes the write validation, which involves sending data to the spi_txfifo via the tl-agent and verifying its subsequent transmission. For read validation, an additional slave device is connected to the SOC. Notably, the exposed SD-related interfaces at the topmost level of the SOC correspond to the four standard SPI interfaces, facilitating the connection with the spi-slave.

For read validation, we refer to the original test cases within the spi-slave, which involve sending 11 bits per transaction with varying commands (occupying the first three bits). Due to our SPI's limitation of transmitting one byte at a time, two data transmissions are required to complete a test. Specifically, data transmitted as A[7:0] and B[7:0] includes command signals in A[6:4], while the actual data consists of {A[3:0], B[7:4]}. Examination of the spi-slave's RTL or TB files reveals support for four commands, with the "read data" operation requiring a third data transmission (of any value) to ensure complete data return via miso. Additionally, each test involves configuring the csmode register to hold, sending data twice to txfifo, and then disabling csmode. Between tests, an additional data transmission to txfifo is necessary to drive the spi-clk and reset certain values within the spi-slave for the next test.

The current write tests cover various polarity, phase, and endianness scenarios using test data from 0x00 to 0xff, all yielding correct results.
### 2.4.3 Hardware Interface   
Within the U500 platform, only a single SPI (SPI0) interface is presently available, specifically designated for connecting to an SD card. The top-level module delineates the following correspondence between the SPI interface signals and the SD card signals:

| SPI Signal | Top-level SD Card Signal | I/O   |
|:------------:|:-------------------------:|:-------:|
| SCK        | sdio_sdio_clk           | output |
| MOSI       | sdio_sdio_cmd           | output |
| MISO       | sdio_sdio_dat_0         | input  |
| CS         | sdio_sdio_dat_3         | output |

#### Detailed Explanation:
- **SCK (Serial Clock)**: The SPI clock signal, utilized for synchronizing data transmission, corresponds to the `sdio_sdio_clk` signal at the top-level module and is an output signal.
- **MOSI (Master Out Slave In)**: The signal for transmitting data from the master device to the slave device. In the context of SD card communication, this represents the master device (e.g., microprocessor) transmitting commands or data to the SD card. It maps to the `sdio_sdio_cmd` signal at the top-level module and is an output signal.
- **MISO (Master In Slave Out)**: The signal for transmitting data from the slave device to the master device. In the context of SD card communication, this represents the SD card returning data or responses to the master device. It maps to the `sdio_sdio_dat_0` signal at the top-level module and is an input signal.
- **CS (Chip Select)**: The signal used to select a specific slave device (SD card) for communication. It maps to the `sdio_sdio_dat_3` signal at the top-level module and is an output signal.


## 2.4.4 Registers    
###  Controller Memory Map Configuration

Offset | Name | Description
:-----------: | :-----------: | :-----------:
0x00 | sckdiv | Serial clock divisor
0x04 | sckmode | Serial clock mode
0x10 | csid | Chip select ID
0x14 | csdef | Chip select default
0x18 | csmode | Chip select mode
0x28 | delay0 | Delay control 0
0x2c | delay1 | Delay control 1
0x38 | extradel |  
0x3c | sampledel |  
  |   |  
0x40 | fmt | Frame format
0x48 | txfifo | Tx FIFO data
0x4c | rxfifo | Rx FIFO data
0x50 | txmark | Tx FIFO watermark
0x54 | rxmark | Rx FIFO watermark
  |   |  
0x60 | fctrl | SPI flash interface control
0x64 | ffmt | SPI flash instruction format
  |   |  
0x70 | ie | SPI interrupt enable
0x74 | ip | SPI interrupt pending

### Register Functionality Overview

#### SckDiv
The sckdiv register specifies the divisor used to generate the serial clock (SCK). The relationship between the input clock and SCK is defined by the following formula:
$$f_{sck}=\frac{f_{in}}{2(div+1)}$$

The input clock is the bus clock tlclk. The div field is reset to the value 0x3.

Alternatively, for a more detailed context:

The input clock is synchronized with the bus clock, denoted as tlclk. The div field within the sckdiv register, which governs the division factor for SCK generation, is initialized to a default value of 0x3 upon reset.

Bits | Fidle Name | Attr. | Rst. | Description
:-----------: | :-----------: | :-----------: | :-----------: | :-----------:
[11:0] | div | RW | 0x3 | Divisor or serial clock
[31:12] | Reserved | RW | X | Reserved


#### Seral Clock Mode Register (sckmode)

The sckmode register dictates the polarity and phase attributes of the serial clock. Upon reset, the sckmode register is initialized to a default value of 0.

Bits | Fidle Name | Attr. | Rst. | Description
:--: | :--: | :--: | :--: | :--:
0 | pha | RW | 0x0 | Serial clock phase
1 | pol | RW | 0x0 | Serial clock polarity
[31:2] | Reserved | RW | X | Reserved

Serial Clock Polarity:  

Value | Description |
:--: | :--: |
0 | The SCK signal remains at a logical level of 0 at idle|
1 |The SCK signal remains at a logical level of 1
at idle|

Serial Clock Phase：  

Value | Description |
:--: | :--: |
0 | Data is sampled on the rising edge of SCK (leading edge) and shifted out on the falling edge of SCK (trailing edge) |
1 | Data is sampled on the falling edge of SCK (leading edge) and shifted out on the rising edge of SCK (trailing edge)|

####  Chip Select ID Register (csid)
The csid register encodes the index of the CS pin to be toggled by the hardware chip select control. The reset value for this register is 0.

Bits | Fidle Name | Attr. | Rst. | Description |
:--: | :--: | :--: | :--: | :--: |
[31:0] | csid | RW | 0x0000_0000 | Chip Select ID |

#### Chip Select Default Register (csdef)
The csdef register specifies the idle state (polarity) of the CS pins. For all implemented CS pins, the reset value is set to high.  

Bits | Fidle Name | Attr. | Rst. | Description |
:--: | :--: | :--: | :--: | :--: |
[31:0] | csdef | RW | 0x0000_0001 | Chip Select Default Value |

#### Chip Select Mode Register (csmode)

The csmode register defines the behavior of the hardware chip select, as detailed in the table below. The reset value is 0 (AUTO). In HOLD mode, the CS pin remains deasserted under the following conditions:

+ A different value is written to either csmode or csid.
+ A write to csdef results in a change of state for the selected pin.
+ Direct-mapped flash mode is enabled.  
    

Bits | Fidle Name | Attr. | Rst. | Description  
:-----------: | :-----------: | :-----------: | :-----------: | :-----------:  
[1:0] | mode | RW | X | Chip Select Mode  
[31:2] | Reserved | RW | X | Reserved
  
  

Value | Name | Description  
:-----------: | :-----------: | :-----------:  
0 | AUTO | Toggle the CS signal at the beginning/end of each frame  
2 | HOLD |Keep the CS signal asserted continuously after the initial frame  
3 | OFF | Disable hardware control over the CS signal


####  Delay Control Registers (delay0 and delay1)

The delay0 and delay1 registers allow the insertion of arbitrary delays in terms of SCK cycles.

- The cssck field specifies the delay between setting CS and the first rising edge of SCK. An additional half-cycle delay is implied when sckmode.pha = 0. The reset value is 0x01.
- The sckcs field specifies the delay between the last falling edge of SCK and the deassertion of CS. An additional half-cycle delay is implied when sckmode.pha = 1. The reset value is 0x01.
- The intercs field specifies the minimum CS idle time between deassertion and reassertion. The reset value is 0x01.
- The interxfr field specifies the delay between two consecutive frames without deasserting CS. This is applicable only when sckmode is in HOLD or OFF. The reset value is 0x00.

delay0：  

Bits | Fidle Name | Attr. | Rst. | Description |
:--: | :--: | :--: | :--: | :--: |
[7:0] | cssck | RW | 0x01 | CS to SCK Delay |
[15:8] | Reserved | RW | X | Reserved |
[23:16] | sckcs | RW | 0x01 | SCK to CS Delay |
[31:24] | Reserved | RW | X | Reserved |

delya1：  

Bits | Fidle Name | Attr. | Rst. | Description |
:--: | :--: | :--: | :--: | :--: |
[7:0] | intercs | RW | 0x01 | Minimum CS inactive time |
[15:8] | Reserved | RW | X | Reserved |
[23:16] | interxfr | RW | 0x00 | Maximum interframe delay |
[31:24] | Reserved | RW | X | Reserved |

####   Frame Format Register (fmt)
The fmt register defines the frame format for transfers initiated via the Programmed I/O (FIFO) interface. The following table describes the proto, endian, and dir fields, which represent the protocol, endianness, and direction, respectively.

- The len field defines the number of bits per frame, with a permissible range from 0 to 8, inclusive.
- For SPI0, the reset value of the fmt register is 0x0008 0008, which corresponds to proto = single, dir = Tx, endian = MSB, and len = 8.
- For SPI1 and SPI2, the reset value of the fmt register is 0x0008 0000, which corresponds to proto = single, dir = Rx, endian = MSB, and len = 8.
    

Bits | Fidle Name | Attr. | Rst. | Description  
:-----------: | :-----------: | :-----------: | :-----------: | :-----------:  
[1:0] | proto | RW | 0x0 | SPI Protocol  
2 | endian | RW | 0x0 | SPI endinanness  
3 | dir | RW | 0x1 | SPI I/O Direction  
[15:4] | Reserved | RW | X | Reserved  
[19:16] | len | RW | 0x8 | Number of bits per frame  
[31:20] | Reserved | RW | X | Reserved


SPI Protocol：  

Value | Description | Data Pins |
:--: | :--: |:--: |
0 | Single | DQ0(MOSI), DQ1(MISO) |
1 | Dual | DQ0, DQ1|
2 | Quad | DQ0, DQ1, DQ2, DQ3 |

SPI Endianness：  

Value | Description |
:--: | :--: |
0 | Transmit most-significant bit (MSB) first |
1 | Transmit least-significant bit (LSB) first |

SPI I/O Direction：

| Value | Description |
|:-------:|:-------------:|
| 0     | Rx: In dual and quad protocols, the DQ pins are tri-stated. In the single protocol, the DQ0 pin is driven by the transmit data as usual. |
| 1     | Tx: Receive data is not stored in the receive FIFO. |

#### Transmit Data Register (txdata)

Writing to the txdata register loads the transmit FIFO with the value in the data field.

For the case where fmt.len < 8, the value should be left-aligned when fmt.endian = MSB, and right-aligned when fmt.endian = LSB.

The full flag is set when the transmit FIFO is ready to accept a new entry; when set, writes to txdata are ignored. On read, the data field returns 0x00.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| [7:0] | data | RW | 0x00 | Transmit Data |
| [30:8] | Reserved | RW | X | Reserved |
| 31 | full | RO | X | FIFO full flag |  

#### Receive Data Register (rxdata)

Reading the rxdata register dequeues a frame from the receive FIFO.

For the case where fmt.len < 8, the value should be left-aligned when fmt.endian = MSB, and right-aligned when fmt.endian = LSB.

The empty flag is set when the receive FIFO contains a new entry available for reading; when set, the data field does not contain a valid frame. Writes to rxdata are ignored.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| [7:0] | data | RO | X | Received Data |
| [30:8] | Reserved | RO | X | Reserved |
| 31 | empty | RO | X | FIFO empty flag |

#### Transmit Watermark Register (txmark)

The txmark register specifies the threshold that triggers the Tx FIFO watermark interrupt.
For spi0, the reset value is 1; for spi1 and spi2, the reset value is 0.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| [2:0] | txmark | RW | 0x1 | Transmit watermark |
| [31:3] | Reserved | RW | X | Reserved |

#### Receive Watermark Register (rxmark)

The rxmark specifies the threshold that triggers the Rx FIFO watermark interrupt. The reset value is 0.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| [2:0] | rxmark | RW | 0x0 | Receive watermark |
| [31:3] | Reserved | RW | X | Reserved |

#### SPI Flash Interface Control Register (fctrl)

When the en bit of the fctrl register is set, the controller enters SPI Flash mode. Access to the memory-mapped region will cause the controller to automatically perform SPI Flash read sequences in hardware. The reset value is 0x1.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| 0 | en | RW | 0x1 | SPI Flash Mode Select |
| [31:1] | Reserved | RW | X | Reserved |

#### SPI Flash Instruction Format Register (ffmt)

The ffmt register defines the format of the SPI Flash read instruction issued by the controller when accessing the memory-mapped region when the controller is in SPI Flash mode.

An instruction consists of a command byte, followed by a variable number of address bytes, dummy cycles (padding), and data bytes. The table below describes the function and reset value of each field.

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| 0 | cmd_en | RW | 0x1 | Enable sending of command |
| [3:1] | addr_len | RW | 0x3 | Number of address bytes (0 to 4) |
| [7:4] | pad_cnt | RW | 0x0 | Number of dummy cycles |
| [9:8] | cmd_proto | RW | 0x0 | Protocol for transmitting command |
| [11:10] | addr_proto | RW | 0x0 | Protocol for transmitting address and padding |
| [13:12] | data_proto | RW | 0x0 | Protocol for receiving data bytes |
| [15:14] | Reserved | RW | X | Reserved |
| [23:16] | cmd_code | RW | 0x03 | Value of command byte |
| [31:24] | pad_code | RW | 0x00 | First 8 bits to transmit during dummy cycles |

#### SPI Interrupt Registers (ie and ip)

The ie register controls which SPI interrupts are enabled, while ip is a read-only register indicating pending interrupt conditions. The reset value of ie is zero.

The txwm condition is triggered when the number of entries in the transmit FIFO is strictly less than the count specified by the txmark register. The pending bit is cleared when the number of enqueued entries is sufficiently large to exceed the watermark.

The rxwm condition is triggered when the number of entries in the receive FIFO is strictly greater than the count specified by the rxmark register. The pending bit is cleared when the number of dequeued entries is sufficiently small to fall below the watermark.

SPI Interrupt Enable Register (ie):  

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| 0 | txwm | RW | 0x0 | Transmit watermark enable |
| 1 | rxwm | RW | 0x0 | Receive watermark enable |
| [31:2] | Reserved | RW | X | Reserved |

SPI Watermark Interrupt Pending Register (ip):  

| Bits | Field Name | Attr. | Rst. | Description |
|:-----:|:-----------:|:------:|:-----:|:------------:|
| 0 | txwm | RO | 0x0 | Transmit watermark pending |
| 1 | rxwm | RO | 0x0 | Receive watermark pending |
| [31:2] | Reserved | RW | X | Reserved |    

## 2.3.5  Checklist