

## i2s

适配麦克风中.

https://www.allaboutcircuits.com/technical-articles/introduction-to-the-i2s-interface/

https://hackaday.com/2019/04/18/all-you-need-to-know-about-i2s/

https://www.jianshu.com/p/e4f07bcd9df4

https://www.cnblogs.com/schips/p/12305649.html

|引脚| 功能|
|---|---|
| SCK/BLCK/SLCK | clock |
| WS/LRCK | word select |
| SD/SDATA | data|
|NC| (悬空) |
|EN| 片选/启用? 直接接到了3v3 |
|LR| 左右选择|

I2S就是被设计来传送音频数据的, 其他的数据都是之后的hacky玩法. 它用一条线区分左右声道, 一条时钟线同步信号, 和一条真正的线传送数据. 在我们板子的receiver=master的情况下, 时钟和WS是接收方发送给麦克风的, 发送方通过SD发送数据给接收方.

I2S允许两个声道的数据在一条线上传送. 因此有了左右声道的选择线. 采样的时候要交替左右轮流读一个字, 导致这个选择线的信号也类似于时钟. 麦克风的规格书里推荐的就是两个麦克风的三条I2S线相连, 一个L/R接地, 一个L/R接电源, 这样就成为了一个立体麦克风.

板子的L/R是接地的, 因此音频要在左声道接受, 需要给出WS为低的时候才有数据, 否则为0. 这是使用的左对齐标准, 24bit的采样数据包装在32位中. 最右边8bit固定为0. Phillips标准则在WS高的时候发送左声道数据.

MSB优先发送. 变化WS之后要等一个时钟周期再开始接受数据. SCLK的频率=2×采样频率×采样位数, LRCK的频率等于采样频率, 这样就刚好能完整收集左右声道的采样数据了.

下面这段来自麦克风的规格书
```
I²S DATA INTERFACE
  The serial data is in slave mode I²S format, which has 24‐bit depth in a 32 bit word. In a stereo frame there are 64 SCK cycles, or 32 SCK cycles per data‐word. When L/R=0, the output data in the left channel, while L/R=Vdd, data in the right channel. The output data pin (SD) is tristated after the LSB is output so that another microphone can drive the common data line.
Data Word Length
  The output data‐word length is 24 bits per channel. The Mic must always have 64 clock cycles for every stereo data‐word (fSCK = 64 × fWS).
Data‐Word Format
  The default data format is I²S, MSB‐first. In this format, the MSB of each word is delayed by one SCK cycle from the start of each half‐frame.
```

## k210的I2S

k210有3个I2S, 因此说它能接6麦克风阵列.

```
其中I²S0 支持可配置连接语音处理模块，实现语音增强和声源定向的功能。
• 总线宽度可配置为8，16，和32 位
• 每个接口最多支持4个立体声通道
• 由于发送器和接收器的独立性，所以支持全双工通讯
• APB 总线和I²S SCLK 的异步时钟
• 音频数据分辨率为12,16,20,24 和32 位
• I²S0 发送FIFO 深度为64 字节, 接收为8 字节，I²S1 和I²S2 的发送和接收FIFO 深度都为8字节
• 支持DMA 传输
• 可编程FIFO 阈值
```

k210使用的是4通道的I2S.

```
[MAIXPY]: numchannels = 2
[MAIXPY]: samplerate = 22050
[MAIXPY]: byterate = 88200
[MAIXPY]: blockalign = 4
[MAIXPY]: bitspersample = 16
```
目前还是没声音, 需要学习I2S的FIFO是什么意思. 深度是什么意思, 然后就是怎么处理ws的, 为什么每个I2S有4个输入,4个输出引脚, 采样率怎么设置

4个输入和4个输出应该是对应4个channel, 可能方便切换吧?? 接受数据的时候, 一个每次传送数据的cycle. 每次传送的数据的bit数

|参数|功能|
|---|----|
|word_length/RESOLUTION|每个word的长度. 12/16/20/24/32选24|
|word_select_size/SCLK_CYCLES|16/24/32选32. 大概是指在WS不变化的时候的cycle数, 也就是WS周期的一半. |
|word_mode|选左对齐. |

有声音了, 关键是上面列举的参数选择. I2S学习先告一段落.

i2s_set_dma_divide_16 函数能设置让DMA的时候自动把32 比特INT32 数据分成两个16 比特的左右声道数据。 那么这32bit的数据从哪来的?

I2S要不要设置时钟周期??