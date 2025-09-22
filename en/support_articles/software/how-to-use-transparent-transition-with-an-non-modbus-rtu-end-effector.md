# How to use the Transparent Transmission Function at the End Effector?  

When the end tool supports RS-485 communication but not the standard Modbus RTU protocol, the end effector transparent transmission function can be used. This function forwards data directly to the end IO board without any data processing.  


## Hardware Connection  
The pin definition diagram of the robotic arm end effector is as follows; the following connections are required:  
![](assets/io.jpg)  

Need to connect 2×24V, 2×GND, RS485A, RS485B.
| Color | Singal      | Color  | Singal  |
| ----- | ----------- | ------ | ------- |
| Brown(PIN1) | +24V(Power) | White(PIN3)  | GND |
| Blue(PIN2)  | +24V(Power) | Green(PIN4)  | GND |
| Pink(PIN5)  | RS485-A     | Yellow(PIN6) | RS485-B |

## Control  
* The end effector baud rate and the end tool baud rate must be consistent. The default end effector baud rate is 2M.  
* The default end effector timeout period is 50ms, which should be adjusted according to the timeout period of the end tool.  


### Control via UFACTORY Studio  
**Version requirement**: ≥V2.7.0;  

Go to the **Settings - External Devices - RS485** interface.  
Optional parameters:  
* RS-485 Port: Select the robotic arm;  
* Baud Rate;  
* Protocol: Select **Transparent Transmission**;  
* Timeout Period;  
![](assets/rs485.png)  

**Note: Click the Save button after modifying the parameters.**  


### Control via Python SDK  

#### 1. Set the Baud Rate  
```python  
code = arm.set_tgpio_modbus_baudrate(2000000)  
```

#### 2. Set the Timeout Period
is_transparent_transmission needs to be set to True (default is False).
```python  
code = arm.set_tgpio_modbus_timeout(timeout=1000, is_transparent_transmission=True)  
```

#### 3. Send Corresponding RS485 Data
is_transparent_transmission needs to be set to True (default is False).
use_503_port needs to be set to True (default uses port 502).

```python  
# Open gripper  
code, ret = arm.getset_tgpio_modbus_data(datas=[0x08, 0x10, 0x07, 0x00, 0x00, 0x02, 0x04, 0x00, 0x00, 0x03, 0x20, 0xFA, 0x2B], is_transparent_transmission=True, use_503_port=True)  
print('open gripper, code={}, ret={}'.format(code, ret))  
time.sleep(0.5)  


# Close gripper  
code, ret = arm.getset_tgpio_modbus_data(datas=[0x08, 0x10, 0x07, 0x00, 0x00, 0x02, 0x04, 0x00, 0x00, 0x00, 0x00, 0xFB, 0x03 ], is_transparent_transmission=True, use_503_port=True)  
print('close gripper, code={}, ret={}'.format(code, ret))  
time.sleep(0.5)  
```