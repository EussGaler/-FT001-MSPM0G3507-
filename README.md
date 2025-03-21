# **[FT001]基于MSPM0G3507的口袋仪器**

<img src="D:\unniversity\EIDC\MSPM0G3507_PocketInstrument\TASK.jpg" style="zoom:50%;" />

FT001--FutureTool未来道具001

---

## **1. 引脚定义**

### **1.1 7针spi OLED屏**


| IO口 | 用途 | 说明 |
| :--: | :--: | :--: |
| PA12 | SCL | GPIO |
| PA13 | SDA | GPIO |
| PA21 | RES | GPIO |
| PA22 | DC | GPIO |
| PA02 | CS | GPIO |

### **1.2 数字示波器**

| 外设/IO口 | 用途 | 说明 |
| :--: | :--: | :--: |
| TIMG0 | 控制ADC采样率 | 初始10us |
| ADC0 | ADC转换 | TIMG0事件触发 |
| DMA_CH0 | 搬运 | ADC触发 |
| PA27 | 模拟信号输入 | ADC |

### **1.3 函数信号发生器**

| 外设/IO口 | 用途 | 说明 |
| :--: | :--: | :--: |
| TIMG12 | 控制DAC更新 | 初始10us |
| DAC0 | DAC转换 | TIMG0中断时更新 |
| PA15 | 波形输出 | DAC |

### **1.4 按键**

| 外设/IO口 |   用途   | 说明 |
| :--: | :------: | :--: |
| PB06 | 按键控制 | GPIO |
| PB07 | 按键控制 | GPIO |
| PA23 | 编码器 | GPIO |
| PA24 | 编码器 | GPIO |

### **1.5 其他**

| 外设/IO口 |   用途   | 说明 |
| :--: | :------: | :--: |
| UART0 | 调试 | 串口通信 |
| PA10 | UART0串口通信 | 开发板TX |
| PA11 | UART0串口通信 | 开发板DX |
| PA14 | 开发板LED上电测试 | GPIO |

## **2. 实现思路**

### **2.1 数字示波器**

1. 使用ADC0的CH0通道，于PA27输入模拟信号。使用单次连续转换，用DMA将数据搬运至大小为1024的数组，取出连续128个数据用于绘制波形。
2. TIMG0发出定时事件触发ADC采样，用于控制ADC采样率。DMA搬运完成（1024次）后触发ADC中断（优先级2），随后更新OLED显示并进行FFT数据处理，延时一段时间方便显示。
3. 通过改变TIMG0自动重装值和绘图时的比例改变水平和垂直档位。
4. FFT的频率精度取决于自动重装值以及采样点数，自动重装值档位$\ge$6时误差$\pm$10Hz；峰峰值误差$\pm$0.2V；~~波形判断精度不太高。~~
5. 示波器触发方式为Roll模式/Y-T模式上升沿触发，Roll模式时不会进行FFT，Y-T模式上升沿触发设定触发电平1.2V，通过在DMA中断中查找触发点实现波形的稳定显示。

### **2.2 函数信号发生器**

1. TIMG12产生优先级0的定时中断更新DAC的输出值（查表），由PA15输出。
2. 输出波形0/1/2/3对应Sin/Squ/Tri/DC。
3. 通过查表时乘一个系数实现输出电压幅值0.5~3.3V的调节（0.5V一档），但这样会加大程序处理时间，故而需要减小TIMG12的中断频率，也会导致波形频率不太精确。
4. 更改TIMG12的自动重装值实现输出频率的调节，共十档（100Hz/200Hz/.../1kHz）。
5. 若频率过高会使程序频繁进中断，严重影响其他模块运行，故而需要TIMG12自动重装值不小于1kU，因此输出不同频率波形时查表使用的数据点的个数不同。

### **2.3 操作指南**

1. PB06按键控制菜单光标移至下一项，PB07按键确认（均为优先级1中断）。
2. 信号发生器页面通过编码器调节Freq，Vpp和Type，通过EXIT确认键返回。示波器页面通过编码器调节X_Lv和Y_Lv，通过确认键调节MODE和EXIT返回。

## **3.更新日志**

### **v1.1.1**

- 优化了示波器水平档位的调节。
  - 现在自动重装值调节范围为399U~6399U，可以更加精细。
  - 屏幕刷新率主要取决于DMA中断中的延时。

### **v1.1.2**

- 加入了FFT并改变了示波器的刻度显示。

### **v1.1.3**

- 从底层改变了波形发生方式，舍弃了SPWM方法，采用了DAC。
  - 旧方法，也有较高精度：~~TIMG7在PA17发出80kHz PWM波，TIMG12每10us产生优先级0的定时中断更新PWM波的占空比，通过外部无源低通滤波获得100Hz，Vpp高于3V的正弦波。~~

### **v1.1.4**

- 修正了OLED闪烁的问题。
- 加入了输出波形幅值和频率的调节。

### **v1.1.5**

- 代码整体规范化。
- 加入了示波器Roll模式和Y-T模式上升沿触发。

### **v1.1.5**

- 加入了串口通信。

### **v1.2.1**

- 加入了二级菜单。
- 将原来的五个按键简化为了两个按键和一个编码器。
- ***版本说明：此版本控制幅值是通过外部分压实现，故而输出波形的精度较高，高于v1.2.2***

### **v1.2.2**

- 加入了软件控制输出波形的幅值。
- ***版本说明：为了实现软件控制幅值而减小了DAC更新频率，故而此版本输出波形的精度较低***

### **下一版本目标**

- 加入Y-T触发电平的编码器调节。

## **4.目前的问题**

1. 烧录时有时不稳定，需要先烧录一个LED闪烁程序覆盖掉原程序。
1. 烧录后需要按一次RST才能正常运行，不过之后上电也能正常运行。
