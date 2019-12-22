# 基于树莓派的智能雨刷器系统
## 简介：
日常生活中，我们经常遇到驾车外出时下雨阻挡视线的情况，这时车窗上的雨刷器就显得格外重要。雨刷器帮助驾驶员清理挡风玻璃上的水珠来保证驾驶员的良好视野进而保障行车安全。然而，雨刷器摆动频率过快和过慢都有负面影响：过快则雨刷器本身会阻挡驾驶员的视野而过慢则难以有效保证挡风玻璃的清洁。于是我便考虑制作一种智能雨刷器来解决这一问题。
## 设计目标：
智能雨刷器所需要实现的目标中最重要的就是保证雨刷器的运转频率适当。也就是根据雨量来自主决定雨刷器的运转频率：__当雨量大时运行频率较高，雨量小时运行频率较低，保证雨刷器运行频率维持在合适的值。__ 
## 设计方案：
为了让系统能得知当前的雨量，一定是需要一个雨量传感器，这里我选择了电压式传感器，它原理与结构都很简单可靠，精度也不错：__两根金属线彼此靠近但不接触，当有雨水落在传感器板子上时，两根金属线导通，产生电压。__  
然后将电压信息传递给 __AD转换器__ 然后将所得数字信息传递给 __树莓派__ 的GPIO接口，由程序读入并分析当前雨量，再输出合适的 __PWM波__ 给控制雨刷器的 __舵机__，最终实现目标。
## 元件选择：
- Raspberry Pi 4B 主板
- 面包板
- AD/DA转换器 PCF8591
- 简易雨滴传感器
- SG-90舵机
- 若干跳线  

Raspberry Pi中文名称为树莓派，不同于单片机，树莓派是一款集成度很高的只有信用卡大小的一个**小型电脑**，电脑能做的它都可以做，不过不同于pc， **树莓派有自带的io口，支持5v和3.33v的输入输出。** 非常好用。  

AD/DA传感器 PCF8591是一款小型的单芯片，单电源和低功耗的8位CMOS数据采集设备，有4个模拟输入，一个模拟输出和一个串行12C总线接口。三个地址引脚用于编程硬件地址，允许最多使用8个连接到12C总线的设备，无需额外的硬件。设备的地址，控制和数据通过双线双向12C总线串行传输。  

雨滴传感器上面已经介绍过了。

SG90舵机是一款小型舵机，广泛用于航模，电子玩具，小型电子仪器以及电子实验中，优点是造价便宜，性能足够强。

## 程序代码：
舵机控制程序：
```python
#!/usr/bin/env python
import RPi.GPIO as GPIO
import time

def rolling(dely = 1):
    #-90度到+90度摆动一个来回
    servopin = 18
    GPIO.setup(servopin, GPIO.OUT, initial=False)
    p = GPIO.PWM(servopin,50) 
    p.start(0)
    
    for i in range(0,181,10):
        p.ChangeDutyCycle(2.5 + 10 * i / 180)   #更改pwm占空比(2.5%-12.5%)设置转动角度  
        time.sleep(0.02)                        #等该20ms周期结束  
        p.ChangeDutyCycle(0)                    #归零信号  
        time.sleep(0.01*dely)
        
    for i in range(181,0,-10):
        p.ChangeDutyCycle(2.5 + 10 * i / 180)
        time.sleep(0.02)
        p.ChangeDutyCycle(0)
        time.sleep(0.01*dely)
    return print('clear');

```

AD转换器控制程序：
```python
import smbus
import time

# for RPI version 1, use "bus = smbus.SMBus(0)"
bus = smbus.SMBus(1)

#check your PCF8591 address by type in 'sudo i2cdetect -y -1' in terminal.
def setup(Addr):
    global address
    address = Addr

def read(chn): #channel
    if chn == 0:
        bus.write_byte(address,0x40)
    if chn == 1:
        bus.write_byte(address,0x41)
    if chn == 2:
        bus.write_byte(address,0x42)
    if chn == 3:
        bus.write_byte(address,0x43)
    bus.read_byte(address) # dummy read to start conversion
    return bus.read_byte(address)

def write(val):
    temp = val # move string value to temp
    temp = int(temp) # change string to integer
    # print temp to see on terminal else comment out
    bus.write_byte_data(address, 0x40, temp)

```

主程序：
```python
#!/usr/bin/env python
import PCF8591 as ADC
import RPi.GPIO as GPIO
import time
import math
import sg90_ctrl as sg90


      #50HZ
DO = 17
GPIO.setmode(GPIO.BCM)

def setup():
    ADC.setup(0x48)
    GPIO.setup(DO, GPIO.IN)

def Print(x):
    if x == 1:
        print('/nnot rain/n')
    if x == 0:
        print('/nrain!/n')

def loop():
    status = 1
    while True:
        tmp = GPIO.input(DO);
        if tmp != status:
            Print(tmp)
            status = tmp
        print(ADC.read(0))
        if ADC.read(0) > 250:
            time.sleep(1)
        elif ADC.read(0) > 200:
            sg90.rolling(5)
        elif ADC.read(0) > 150:
            sg90.rolling(4)
        elif ADC.read(0) > 100:
            sg90.rolling(3)
        elif ADC.read(0) > 50:
            sg90.rolling(2)
        elif ADC.read(0) > 0:
            sg90.rolling(1)
        

if __name__ == '__main__':
    try:
        setup()
        loop()
    except KeyboardInterrupt: 
        pass    

```
## 遇到的问题：
树莓派入门调试时遇到不少麻烦这里不一一列举了。  
一开始时对如何模拟雨刷器行为比较困惑，经过调查后发现用SG90小型舵机来模拟雨刷器比较合理，但是如何实现不同频率下的雨刷器行为。一开始以为需要DA转换器，但是我只有一个PCF8591用于雨滴传感器。后来我了解到SG90本身支持PWM波，于是便写了一个py程序控制GPIO口输出PWM波来控制舵机，效果很理想。

## 心得体会：
树莓派真的很好用，对比于以前学的51单片机使用汇编语言编程和后来学的FPGA的HDL语言，树莓派的Python3简直是**天堂**，逻辑处理交给cpu，电路只负责接受输入信号，对任何人来说都 <big>**非常友好**</big>。  
另外本来上这门课之前感觉传感器是很难理解的东西，但是经过这个项目之后感觉传感器还是很亲民的。  
最后，**树莓派真的很棒**。
