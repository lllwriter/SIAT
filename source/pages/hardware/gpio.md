# 2.2 GPIO  
## 2.2.1 GPIO Technical Specification  

![alt text](photo/image-26.png)  

### Description
The GPIO module facilitates software communication through general-purpose I/O pins with a high degree of flexibility. Each of the 32 individual bits can be configured as peripheral outputs using two distinct modes. Similarly, each of these 32 bits can be accessed by software as peripheral inputs. The connectivity of these peripheral inputs and outputs to the chip's I/O pins is beyond the scope of this document. Refer to the Comportability Specification for detailed peripheral I/O options at the top chip level.

In the output configuration, this module offers direct 32-bit access to each GPIO value via direct write functionality. This mode empowers software to control all GPIO bits simultaneously. Alternatively, the module supports masked writes, which allow software to update half of the bits at once, enabling the modification of a subset of the output values without necessitating a read-modify-write sequence. In this mode, the user specifies a mask indicating which of the 16 bits are to be altered, along with their new values. Detailed operation of this mode is elucidated in the Programmers Guide.

For input operations, software can read the status of any GPIO peripheral input. Additionally, software can configure interrupt events for any of the 32 bits, with options including positive edge, negative edge, or level detection. A noise filter can be enabled for any of the 32 GPIO inputs, requiring the input to remain stable for 16 module clock cycles before the input register acknowledges the change and evaluates interrupt generation. It is important to note that if the filter is activated and the pin is designated as an output, there will be a corresponding delay in reflecting output changes in the input register.

For in-depth insights into output, input, and interrupt control mechanisms, please consult the Design Details section.
### Features
- 32 General Purpose Input/Output (GPIO) ports
- Configurable interrupt capability for each GPIO, supporting rising edge, falling edge, or active low/high input detection
- Dual methods for updating GPIO output: direct write and masked (thread-safe) update   
  
## 2.2.2 Theory of Operation    
### GPIO Output
The GPIO module maintains a single 32-bit output register, DATA_OUT, with two methods of write access. Direct write access utilizes DIRECT_OUT, while masked write access employs MASKED_OUT_UPPER and   
  
MASKED_OUT_LOWER. Direct access allows full read and write capabilities for all 32 bits within one register.

For masked access, the bits to be modified are specified as a mask in the upper 16 bits of the MASKED_OUT_UPPER and MASKED_OUT_LOWER register writes, with the data to be written provided in the lower 16 bits of the register write. The hardware updates DATA_OUT according to this mask, enabling modifications without requiring software to perform a Read-Modify-Write operation.

Reads from masked registers return the lower or upper 16 bits of the DATA_OUT contents, with zeros returned in the upper 16 bits (mask field). To read the values on the pins, software should access the DATA_IN register. (Refer to the GPIO Input section below).

This same concept is applied to the output enable register, DATA_OE. Direct access uses DIRECT_OE, and masked access is facilitated through MASKED_OE_UPPER and MASKED_OE_LOWER. The output enable is transmitted to the pad control block to determine whether the pad should drive the DATA_OUT value to the associated pin.

A typical usage pattern involves initializing and suspending/resuming code utilizing the full access registers to set the output enables and current output values, before switching to masked access for both DATA_OUT and DATA_OE.

For GPIO outputs that are not in use (either not connected to a pin output or not selected for pin multiplexing), the output values are disconnected and have no impact on the GPIO input, regardless of the output enable settings.

### GPIO Input
The DATA_IN register provides the contents as observed on the peripheral input, usually from the pads connected to these inputs. In the presence of a pin-multiplexing unit, GPIO peripheral inputs not connected to a chip input will be tied to a constant zero input.

The GPIO module offers optional independent noise filter control for each of the 32 input signals. Each input can be independently enabled with the CTRL_EN_INPUT_FILTER (one bit per input). This 16-cycle filter is applied to both the DATA_IN register and the interrupt detection logic. The timing for DATA_IN remains non-instantaneous if CTRL_EN_INPUT_FILTER is false due to top-level routing, but no flops are present between the chip input and the DATA_IN register.

The contents of DATA_IN are always readable and reflect the value seen at the chip input pad, irrespective of the output enable setting from DATA_OE. If the output enable is true (and the GPIO is connected to a chip-level pad), the value read from DATA_IN includes the effect of the peripheral's driven output (thus differing from DATA_OUT only if the output driver is unable to switch the pin or during the delay imposed if the noise filter is enabled).

### Interrupts
The GPIO module provides 32 interrupt signals to the main processor. Each interrupt can be independently enabled, tested, and configured. Following the standard interrupt guidelines in the Comportability Specification, the 32 bits of the INTR_ENABLE register determine whether the associated inputs are configured to detect interrupt events. If enabled via the various INTR_CTRL_EN registers, their current state can be read in the INTR_STATE register. Clearing is achieved by writing a 1 into the associated INTR_STATE bit field.

For configuration, four types of interrupts are available per bit, controlled with four control registers. INTR_CTRL_EN_RISING configures the associated input for rising-edge detection. Similarly, INTR_CTRL_EN_FALLING detects falling edge inputs. INTR_CTRL_EN_LVLHIGH and INTR_CTRL_EN_LVLLOW allow the input to be level-sensitive interrupts. Theoretically, an input can be configured to detect both rising and falling edges, but there is no hardware assistance to indicate which edge caused the output interrupt.

**Note #1**: The interrupt can only be triggered by GPIO input. **Note #2**: All inputs are subject to optional noise filtering before being sent into interrupt detection. **Note #3**: All interrupts to the processor are level interrupts as per the Comportability Specification guidelines. The GPIO module, if configured, converts an edge detection into a level interrupt to the processor core.

