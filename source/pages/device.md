
# 5. DEVICE(SD)
##  5.1 SD Technical Specification  
### Description
SD memory card is a new generation of memory device based on semiconductor flash memory, which is widely used in portable devices, such as digital cameras, smart phones and multimedia players, due to its excellent features such as small size, fast data transfer speed and hot-swappable.

SD cards have two communication modes, sdio and spi. The SPI mode consists of an auxiliary communication protocol provided by flash-based SD memory cards. This mode is a subset of the SD memory card protocol, and the interface is selected during the first reset command (CMD0) after power-on and cannot be changed once the device is powered on.

In the case of standard-capacity memory cards, data blocks can be as large as one card write block and as small as one byte. The card option specified in the CSD register enables partial block read/write operations.

In the case of high-capacity SD memory cards, the data block size is fixed at 512 bytes. the block length set by CMD16 is used only for CMD42 and not for memory data transfer. Therefore, partial block read/write operations are also disabled. In addition, write-protect commands (CMD28, CMD29 and CMD30) are not supported.

###  Features
- SD card verilog simulation model with spi mode (mode 0) support
- High-capacity SD memory card (SDHC) with a size of 32GB
- Operating voltage range of 2.7-3.6V
- Support card initialization, write data, read data
- Support commands including CMD0, CMD8, CMD12, CMD16, CMD17, CMD18, CMD24, CMD25, CMD55, CMD58, ACMD41 and so on.

## 5.2 Theory of Operation

###  SPI Mode

SPI messages consist of commands, responses, and data block tokens. All communication between the host and the card is controlled by the host. The host initiates each bus process by setting the CS signal low.

The selected card always responds to commands, and when the card encounters a data retrieval problem during a read operation, it responds with an error response (replacing the expected block of data) rather than a timeout as in SD mode.

In addition, each data block sent to the card during a write operation will respond with a data response token.

### Power-on

The state machine st waits 74 clock cycles in the PowerOn state and switches to the IDLE state when power-on is complete.

### Receiving Commands

When a 48-bit command is transmitted on the mosi line, he first two bits 0x01 of the command is recognised and the state machine st switches from IDLE to CmdBit46 and then to CommandIn to receive the next 46 bits of data.

###  Responding to Commands

The state machine st receives the command in the CommandIn state and switches to CardResponse state, in which it responds to the command invocation function and the response is sent to the miso line.

After the response is sent the state machine st jumps to the corresponding state to complete the wait or read/write data operation according to the previously received command.

###  Card Initialisation

Use state machine ist to respond to initialisation commands and set init_done to 1 when the commands have all completed.

###  Clock and Phase

A maximum clock of 400KHz is required in Card Idntification Mode.
A maximum clock of 25MHz is required in Data Transfer Mode.

The SPI device module only internally supports mode 0, where data is shifted out on the falling edge and sampled on the rising edge.

The SPI device module internally supports mode 0, in which data is shifted out on the falling edge and sampled on the rising edge, and the SPI clock returns high at the end of the transaction.

## 5.3 SD Card Design Verification
**Goals**

 Verify SD card features by running dynamic simulations with a SV/UVM based testbench    

**Design features**   

For detailed information on UART design features, please see the **SD Technical   Specification**.      

**Testbench architecture**

The top-level testbed is located in ‘\trunk\hw\d2dv100_top\dv’ and the SD card model is instantiated in dut_MEISHAV100_TOP_wrapper.sv.

| SD card signals | SD card signals given by the top-level module |  I/O   |
| :-------------: | :-------------------------------------------: | :----: |
|      sclk       |                    spi_clk                    | output |
|      mosi       |                     MOSI                      | output |
|      miso       |                     MISO                      | input  |
|       ncs       |                     SS_n                      | output |
|      rstn       |                   spi_rst_n                   | output |

**Stimulus strategy**

This test was modified based on the sd.c code written for the Freedom U500.

**Self-checking strategy**

Provide the spi port on the soc to send data to the spi-slave to see if the spi-slave can send back the correct value.

In the main function, send data to spi_txfifo via REG32(spi, SPI_REG_TXFIFO), then the spi will automatically send the data from the txfifo, and we only need to check if the mosi port is correctly outputting in the monitor. Since the txfifo can only send one byte of data at a time, choosing 0x00~0xff as the test data will cover all cases.

### 5.3.1 Testplan
#### 5.3.1.2 SD card initialisation test
The SD card needs to be initialised before performing data read and write operations.
 **Power-on and clock pulse sending**:  

   - After the SD card is powered on, the host needs to send at least 74 clock pulses to the SD card. This is because the SD card needs about 64 clock cycles to reach the normal operating voltage, after which 10 clock cycles are used to synchronise with the SD card.
   - During this time, the chip select signal ncs and the master output slave input signal mosi should be held high.  
    
**Send CMD0 command**:  

   - The chip select signal ncs is pulled low and the CMD0 (0x40) command is sent to the SD card with parameters 0x00_00_00_00_00 and CRC check bit 0x95.
   - This command is used to reset the SD card. After the command is sent, wait for the SD card to return the response data R1.  
    
**Check CMD0 response:**  

   - Check the returned response data R1. If R1 is 0x01, it indicates that the SD card is in idle state and you can enter SPI mode and continue to the next step. If R1 is any other value, you need to re-execute step 2.
   - Wait for 8 clock cycles and then pull up the chip select signal ncs.  
    
**Send CMD8 command**:  

   - Pull down the chip select signal ncs again and send CMD8 (0x48) command with parameter 0x00_00_01_AA.
   - This command is used to query the version number of the SD card. After the command is sent, wait for the SD card to return the response data R7.  
 
**Check CMD8 response**:  

   - Check the returned response data. If R1 is 0x05, the SD card is not an SD 2.0 card. If R1 is 0x01 and the 32-bit parameter is 0x00_00_01_AA, it is judged to be an SD 2.0 card. If the returned data is incorrect, step 4 needs to be performed again.
   - Wait for 8 clock cycles and pull up the chip select signal ncs.  
      
  
**Sends APP_CMD (CMD55), the precursor command to the ACMD41 command**:  
   - Pull down the chip select signal ncs and send CMD55 (0x77) command to inform the SD card that the next send is a application-specific command.
   - Wait for the SD card to return the response data R1, check if R1 is 0x01. if yes, continue to the next step; if no, re-execute step 6.  
      
**Send ACMD41 command**:  

   - Pull down the chip select signal ncs and send the ACMD41 (0x69) command to query the SD card initialisation result.
   - Wait for SD card to return response data R1.  
  
**Check ACMD41 response**:  

   - Check R1 in the returned response data R3. if R1 is 0x00, initialisation is complete. If not, it is necessary to re-execute step 6 until the initialisation is successful.
   - Wait for 8 clock cycles and then pull up the chip select signal ncs.

The above steps outline the initialisation process for SD 2.0 specification SD cards, ensuring that the SD card has been correctly initialised prior to data reading and writing.3.5.2

#### 5.3.1.3 SD card data write test

After the initialisation of the SD card is completed, the data writing operation is performed first, and then the written data is read out and verified.

##### 5.3.1.3.1 Single Block Write Operation  
  
 **Send Write Command**:
   - Pull down the chip select signal ncs.
   - Send the CMD24 command (0x58) to the SD card, carrying the 4-byte SD card write sector address as a parameter.
   - The CRC check byte is not used and is written directly to 0xAA.
   - After the command is sent, wait for the SD card to return the response data.  
    
**Check response and write data**:
   - If the SD card returns the correct response data R1 is 0x00, wait 8 clock cycles.
   - Write token 0xFE to the SD card, immediately followed by a 512-byte block of data.  
  
**Write CRC**:
   - In SPI mode, no CRC check is performed on the data, and two bytes of 0xFF are written directly as the CRC check byte.  
  
**Waiting for SD card response**:
   - After the verification data is sent, the SD card returns the response data.
   - The SD card then pulls the MISO signal low and enters the write busy state.  
  
**Write operation complete**:
   - Wait for the MISO signal to be pulled high again to indicate that the SD card exits the write busy state.
   - Wait for 8 clock cycles and then pull up the chip select signal ncs.
   - At this point, the SD card data write operation is complete and other operations can be performed.

##### 5.3.1.3.2 Multiple Block Write Operation

**Send write command**:
   - Pull down the chip select signal ncs.
   - Send a CMD25 command (0x59) to the SD card, carrying the 4-byte SD card write sector address as a parameter.
   - The CRC check byte is not used and is written directly to 0xAA.
   - After the command is sent, wait for the SD card to return the response data.  
    
 **Check response and write data**:
   - If the SD card returns the correct response data R1 is 0x00, wait for 8 clock cycles.
   - Write token 0xFC to the SD card, immediately followed by a 512-byte block of data.  
   
 **Write CRC check byte**:
   - In SPI mode, no CRC check is performed on the data, and two bytes of 0xFF are written directly as the CRC check byte.  
  
**Wait for SD card response**:
   - After the check data is sent, the SD card will return the response data.
   - The SD card then pulls the MISO signal low and enters the write busy state.  
  
 **Continue to send data block**:
   - Wait for the MISO signal to pull high again to indicate that the SD card exits the write busy state.
   - Write token 0xFC to the SD card, immediately followed by a 512-byte data block.  

**Write CRC check byte**:
   - In SPI mode, no CRC check is performed on the data, and two bytes of 0xFF are written directly as the CRC check byte.  
  
 **Wait for SD card response**:
   - After the check data is sent, the SD card will return the response data.
   - The SD card then pulls the MISO signal low and enters the write busy state.  
    
**Write operation complete**:
   - Wait for the MISO signal to pull high again to indicate that the SD card exits the write busy state.
   - Send the write end token 0xFD.
   - Wait for 8 clock cycles and then pull up the chip select signal ncs.
   - At this point, the SD card data write operation is complete and other operations can be performed.

The above steps outline the process of SD card data writing to ensure that the data can be correctly written to the specified sectors of the SD card, and to perform the necessary status checks after the writing is complete.

#### 5.3.1.4 SD card data reading test

After the SD card has been initialised and data written, the operation of reading and verifying the data is performed next.

#### 5.3.1.4.1 Single Block Read Operation

 **Send read command**:
   - Pull down the chip select signal ncs.
   - Send the CMD17 command (0x51) to the SD card, carrying the 4-byte SD card read sector address as a parameter.
   - The CRC check byte is not used and is written directly to 0xFF.
   - After the command is sent, wait for the SD card to return the response data.  
      
 **Check the response and read the data**:
   - If the SD card returns the correct response data R1 is 0x00, prepare to receive the data using the data header 0xFE returned by the SD card as a flag.
   - Receive 512 bytes of data read from SD card and 2 bytes of CRC check byte.  
    
 **Parsing data**:
   - After confirming data token 0xFE, receive 512 bytes of data returned from SD card.
   - After data parsing is completed, receive two bytes of CRC check value. In SPI mode, no CRC check is performed on the data, so these two bytes can be ignored.  
   
**Read operation complete**:
   - After the CRC check byte is received, wait for 8 clock cycles.
   - Pulling up the chip select signal ncs marks the completion of a data read operation.

#### 5.3.1.4.2 Multiple Block Read Operation

  
**Send read command**:
   - Pull down the chip select signal ncs.
   - Send the CMD18 command (0x52) to the SD card, carrying the 4-byte SD card read sector address as a parameter.
   - The CRC check byte is not used and is written directly to 0xFF.
   - After the command is sent, wait for the SD card to return the response data.
  
 **Check the response and read the data**:
   - If the SD card returns the correct response data R1 is 0x00, prepare to receive the data using the data token 0xFE returned by the SD card as a flag.
   - Receive 512 bytes of data read from SD card and 2 bytes of CRC check byte.
  
 **Parsing data**:
   - After confirming data token 0xFE, receive 512 bytes of data returned from SD card.
   - After data parsing is completed, receive two bytes of CRC check value. In SPI mode, no CRC check is performed on the data, so these two bytes can be ignored.
  
**Interrupt reading**:
   - Send the abort command cmd12 (0x4c), the abort command can be sent at any time, when 512 bytes are sent, stop sending data.
   - If cmd12 is not sent, it will continue to read the next data block.
  
 **Read operation completed**:
   - After the CRC check byte is received, wait for 8 clock cycles.
   - Pull up the chip select signal ncs to mark the completion of a data read operation.

The above steps ensure that the previously written data is correctly read from the SD card and the necessary verification is performed.

## 5.4 Hardware Interfaces

### 5.4.1 SD card I/O

| Pin name | Direction |                       Description                        |
| :------: | :-------: | :------------------------------------------------------: |
|   sclk   |   input   |                      SD Card Clock                       |
|   ncs    |   input   |                    chip select signal                    |
|   mosi   |   input   |  SD card input, host sends commands and data to SD card  |
|   miso   |  output   | SD card output to send data or response to host computer |

## 5.5 Registers

|   Name    | Offset | bit width |                         Description                          |
| :-------: | :----: | :-------: | :----------------------------------------------------------: |
|    csd    |   \    |    128    | The CSD is card-specific data register that provides information. |
|    cid    |   \    |    128    | The CID (Card Identification) register contains the card identification information. The value of the CID register is vendor specific. |
|    ocr    |   \    |    32     | This register describes the operating voltage range and status bit in the power supply.This model is SDHC with a voltage range of 2.7-3.6V |
|    scr    |   \    |    64     | SCR (SD Card Configuration Register) provides information on the SD memory card’s special features. |
|    st     |  0x0   |     4     |               Record SD card state transitions               |
|    ist    |  0x0   |     3     |  Recording state transitions during SD card initialisation   |
| init_done |  0x0   |     1     |                Initialisation completion flag                |
|   token   |  0x0   |     8     |          Record the start token for receiving data           |
| multi_st  |  0x0   |     2     | State machine for continuous data writing, implementation of interrupt commands |

#### two-dimensional array

|   Name    | bit width |      bit depth       |                       Description                        |
| :-------: | :-------: | :------------------: | :------------------------------------------------------: |
| flash_mem |     8     | 32\*1024\*1024\*1024 | Simulates the data storage area of an SD card, size 32GB |

## 5.6 Checklist

### Design

| Command   | Result || Command   | Result || Command   | Result |
| :---------: | :------:|| :---------: | :------:|| :---------: | :------:|
| cmd0      | done   || cmd18     | doen   || cmd42     | waived |
| cmd1      | waived || cmd24     | done   || cmd56     | waived |
| cmd6      | waived || cmd25     | done   || cmd58     | done   |
| cmd8      | done   || cmd27     | waived || cmd59     | waived |
| cmd9      | done   || cmd28     | waived || acmd13    | done   |
| cmd10     | done   || cmd29     | waived || acmd18    | waived |
| cmd12     | done   || cmd30     | waived || acmd22    | waived |
| cmd13     | waived || cmd32     | waived || acmd23    | waived |
| cmd16     | done   || cmd33     | waived || acmd25    | waived |
| cmd17     | done   || cmd38     | waived || acmd26    | waived |  
| acmd38    | waived || acmd41    | done   || acmd42    | waived |
| acmd43-49 | waived || acmd51    | done   |

### Verify

| Command   | Result    || Command   | Result    || Command   | Result    |
| :---------: | :---------: || :---------: | :---------: || :---------: | :---------: |
| cmd0      | done      || cmd27     | waived    || acmd18    | waived    |
| cmd1      | waived    || cmd28     | waived    || acmd22    | waived    |
| cmd6      | waived    || cmd29     | waived    || acmd23    | waived    |
| cmd8      | done      || cmd30     | waived    || acmd26    | waived    |
| cmd9      | not start || cmd32     | waived    || acmd38    | waived    |
| cmd10     | not start || cmd33     | waived    || acmd41    | done      |
| cmd12     | done      || cmd38     | waived    || acmd42    | waived    |
| cmd13     | waived    || cmd42     | waived    || acmd43-49 | waived    | 
| cmd16     | done      || cmd55     | done      || acmd51    | not start |
| cmd17     | done      || cmd56     | waived    || acmd13    | not start | 
| cmd18     | doen      || cmd58     | done      || cmd25     | done      | 
| cmd24     | done      || cmd59     | waived    |













