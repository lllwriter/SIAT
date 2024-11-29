# 4 Backend   
## 4.1 Memory
### Description

A shell script has been developed to automatically generate the required memory using the memory compiler tool by filling in the parameters according to the example table.
### Features

- Developed a script that automatically generates the required memory using the memory compiler and captures the relevant parameter information to generate a report.
- IP using UMC 40nm
- Memory selection focuses on performance (read cycles) and area



### 4.1.1 Theory of Operation

#### 4.1.1.1 Script Functions

- Automated memory generation
- Captures memory parameters, analyses them and generates a report.

### 4.1.2 Environmental requirements

#### Operating system version

- Linux: any modern Linux distribution, such as Ubuntu 20.04 LTS, Fedora 34, CentOS 7, etc.
- macOS: macOS 10.15 Catalina or later.
- Other Unix-like systems: Any Unix-like system that supports Bash or other compatible shells.

#### Programming language version

- Shell: Bash 4.0 or higher (the default version of Bash installed on most modern Linux distributions meets this requirement).

#### Other dependencies

- A memory compiler with command line generation support.

### 4.1.3 Usage Guidelines

#### 4.1.3.1 Fill in the parameters

Enter the parameters in the input_data.csv file (in the following format).

| proc  | type | word | bit  | byte | ringtype | cksr | datasr | load |
| ----- | ---- | ---- | ---- | ---- | -------- | ---- | ------ | ---- |
| fsh0l | 1P   | 64   | 21   | 4    | ringless | 0.4  | 0.4    | 0.3  |
| fsh0l | 1P   | 8192 | 8    | 8    | ringless | 0.4  | 0.4    | 0.3  |

#### 4.1.3.2 Modify weights

Modify the time and area weights in the script file 'run_allrun.sh'. This weight affects the scoring of the instance and is a key metric for picking the best memory. The time weight is 'taa_weight' and the area weight is 'area_weight'.

#### 4.1.3.3 Running Scripts

Run the ‘. /run_allrun.sh’ command to generate the script automatically.

### 4.1.4 Report Output

#### 4.1.4.1 txt file

- The generated report is in txt format.
- The report contains the length, width, area and read cycles of the memory.
- The report also contains the overall rating of each instance and the best rated instance.

#### 4.1.4.2  Verilog files and library files

- Automatically generates verilog and library files based on the best rated instances in the txt report.





## 4.2 I/O Cells Selections

MEISHAV100 adopts the UMC 40 nm I/O cell library. I/O cell types are selected based on specific functional characteristics and alignment with project requirements. A final section provides the pin usage of the selected I/O cells.

### 4.2.1 I/O Cell Type Selection

I/O cells are categorized into the following types:

- analog I/O cells
- the crystal oscillator I/O cells
- tolerance generic POC I/O cells
- supports the speed requirement of the DDR2 SDRAM interface I/O cells
- supports SSTL-18,MDDR (LPDDR) and SSTL-2 modes I/O cells
- multi-voltage generic POC I/O buffer cells

This project selects I/O cells that support DDR2 SDRAM interface speeds and multi-voltage general-purpose POC I/O buffer units.

### 4.2.2 Selection of Specific I/O Cells

The I/O cell types were determined based on pin frequency and functional requirements of the project. Pin frequencies range from 25 MHz to 300 MHz, with the need for differential clocking. Ultimately, three I/O cells were selected.

The table below lists the I/O cells selected for this project.

| cell name                                                    | function                                              | sizes               | voltage&frequency                            |
|:-------------------------: | :----------------:| :---------: |:-------------:|
| ZTRNE4GS<br />(FOH0L_QRS25_TMVH33L18<br />_GENERIC_POC_IO CELL LIBRARY) | Programmable 4.0 mA ~ 16 mA CMOS bidirectional buffer | 24.92μm x 169.4μm   | 3.3V:150MHz<br />2.5V:100MHz<br />1.8V:80MHz |
| WOSSTLECDLK<br />(FOH0L_QRS25_T18<br />_SSTL18A_IO CELL LIBRARY) | Differential bidirectional buffer                     | 70 µm x 252.98 µm   | 1.8v:supports DDR2 SRAM interface            |
| WRNE4GSLAX250<br />(FOH0L_QRS25_T18_GENERIC<br />_POC_IO_250 CELL LIBRARY SPECIFICATION) | Programmable 4.0 mA ~ 16 mA CMOS bidirectional buffer | 29.96 μm x 169.4 μm | 1.8v:250Mhz                                  |

### 4.2.3 The truth table for I/O selection

- **ZTRNE4GSLA(WRNE4GSLAX250)**

The table below summarizes the function and specific values for each pin

| E    | Output   |
| :----: | :--------: |
| 0    | disabled |
| 1    | enabled  |

| IE   | Input    |
| :----: | :--------: |
| 0    | disabled |
| 1    | enabled  |

| SR   | Slew rate |
| :----:| :---------: |
| 0    | fast      |
| 1    | slow      |

| SMT  | input           |
| :----: | :---------------: |
| 0    | normal          |
| 1    | schmitt-trigger |

| PU   | PD   | Pull-up/Pull-down       |
| :----: | :----: | :-----------------------: |
| 0    | 0    | None                    |
| 1    | 0    | 75-kΩ pull-up           |
| 0    | 1    | 75-kΩ pull-down         |
| 1    | 1    | keeper at IE=1          |
| 1    | 1    | 75-kΩ pull-down at IE=0 |

| E8   | E4   | driving capability |
| :----: | :----: | :------------------: |
| 0    | 0    | 4*X                |
| 0    | 1    | 8*X                |
| 1    | 0    | 12*X               |
| 1    | 1    | 16*X               |

The following table represents the truth table for the selected I/O types

| I    | E    | IO          | IE   | O    |
| :----: | :----: | :-----------: | :----: | :----: |
| 0    | 1    | 0           | 1    | 0    |
| 1    | 1    | 1           | 1    | 1    |
| 0    | 1    | 0           | 0    | 0    |
| 1    | 1    | 1           | 0    | 0    |
| X    | 0    | Z           | 1    | X    |
| X    | 0    | 1           | 1    | 1    |
| X    | 0    | 0           | 1    | 0    |
| X    | 0    | pull-down   | 1    | 0    |
| X    | 0    | pull-uo     | 1    | 1    |
| X    | 0    | none/keeper | 1    | IO   |
| X    | 0    | X           | 0    | 0    |

- **WOSSTLECDLK**

The table below summarizes the function and specific values for each pin

| 信号        | 功能                                                         |
| :-----------: | :-----------------------: |
| I           | Non-inverting input signal of the driver                     |
| E           | Output enable（‘1’enabled）                                  |
| IO          | Input pad of the receiver/Output pad of the driver           |
| IOB         | Inverted input pad of the receiver/Inverted output pad of the driver |
| E3V         | VCCK power-down detection pin（‘1’enabled）                  |
| SIO         | Single-ended mode/Differential mode selection pin（‘0’differential input mode/differential output mode） |
| NLEG5~NLEG0 | NMOS driving compensation control pins（Typ.default setting：010100） |
| PLEG5~PLEG0 | PMOS driving compensation control pins（Typ.default setting：010100） |
| REF         | Reference voltage of the single-ended receiver               |
| IE          | Input enable pin（‘1’enabled）                               |
| O           | Output signal of the receiver                                |
| RONMD2~0    | Output resistance selection pin                              |
| ODTEN       | ODT enable pin（‘1’enabled）                                 |
| ODTMD2~0    | ODT resistance selection                                     |
| D15V        | I/O power selection pin（’0‘：1.2v or 1.35v     ’1‘：1.5v or 1.8v） |
| MDDR1       | MDDR mode selection pin（’0’：DDR2 mode    ‘1’：LPDDR(MDDR)mode） |

the selection of the output driver impedance

| RONMD2 | RONMD1 | RONMD0 | Description |
| :------: | :------: | :------: | :-----------: |
| 0      | 0      | 0      | 120Ω        |
| 0      | 0      | 1      | 120Ω        |
| 0      | 1      | 0      | 80Ω         |
| 0      | 1      | 1      | 60Ω         |
| 1      | 0      | 0      | Reserved    |
| 1      | 0      | 1      | 48Ω         |
| 1      | 1      | 0      | 40Ω         |
| 1      | 1      | 1      | 34.3Ω       |

On-die termination

| ODTMD2 | ODTMD1 | ODTMD0 | ODTEN | Description     |
| :------: | :------: | :------: | :-----: | :---------------: |
| X      | X      | X      | 0     | ODT is disabled |
| 0      | 0      | 0      | 1     | ODT is disabled |
| 0      | 0      | 1      | 1     | 120Ω            |
| 0      | 1      | 0      | 1     | 60Ω             |
| 0      | 1      | 1      | 1     | 40Ω             |
| 1      | 0      | 0      | 1     | Reserved        |
| 1      | 0      | 1      | 1     | 30Ω             |
| 1      | 1      | 0      | 1     | 24Ω             |
| 1      | 1      | 1      | 1     | 20Ω             |
  

- **XOSCAHINTLA**

The table below summarizes the function and specific values for each pin

| Control Pin | Description                        |
| ----------- | ---------------------------------- |
| S0 and S1   | Output driving capability control  |
| E and EB    | Operating mode control             |
| FEB         | Internal feedback resistor control |

Programmable Features of Output Driving Capability

| S0   | S1   | Programmable Feature                        |
| ---- | ---- | ------------------------------------------- |
| 0    | 0    | For operating frequency at 1.0 MHz ~ 12 MHz |
| 0    | 1    | For operating frequency at 12 MHz ~ 24 MHz  |
| 1    | 0    | For operating frequency at 24 MHz ~ 36 MHz  |
| 1    | 1    | For operating frequency at 36 MHz ~ 48 MHz  |

Programmable Features of Operating Mode

| E    | EB   | Programmable Feature   |
| ---- | ---- | ---------------------- |
| 0    | 0    | Power-down (Stop) mode |
| 0    | 1    | Parallel mode          |
| 1    | 0    | Oscillation mode       |
| 1    | 1    | Tri-state mode         |

Programmable Features of Internal Feedback Resistor

| FEB  | Programmable Feature          |
| ---- | ----------------------------- |
| 0    | With the internal feedback    |
| 1    | Without the internal feedback |

