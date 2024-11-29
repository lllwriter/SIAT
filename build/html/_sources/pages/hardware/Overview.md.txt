# 2.1 OVERVIEW  
![alt text](photo/image-26.png)    
   
<div align=center> 
Completion 
<div>     

<div align=left> 
<div>    

The table below provides a list of the hardware modules we currently possess:   

|Design|Spec Version|    
|:---:|:---:|  
|GPIO|1.1.0|  
|SPI|1.1.0|  
|QSPI|1.1.0|  
|UART|1.1.0|  
|JTAG|1.1.0|  
|SRAM|1.1.0|         

+ The chip incorporates GPIO peripherals, which facilitate bidirectional communication with the external environment.
+ The chip features two SPIs and one QSPI, supporting both single-wire and four-wire modes, and is employed for communication with external devices, such as SD cards.
+ The chip includes a UART peripheral, which implements dual-lane full-duplex UART functionality.
+ The chip houses a JTAG port. By interfacing with the JTAG pins, the logic within the debug module enables the core to enter debug mode, affording the ability to inject code either into the device—through instruction emulation—or directly into memory. 