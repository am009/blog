# SPI

Widora AIRV R3 适配Maixpy 项目

## 前言

看到推荐就买了, 毕竟板载的传感器太全了, 一个小板子就什么都有了, 而且便宜. 导致我走上了这条路. 希望能静下心来. 学到什么. 我经常为了像这样做到些什么而学习, 但往往走偏, 后期只想着想要做到的事情, 完全抛弃了学习, 瞎试, 就是不去继续学.

TODO: 为maixpy 贡献一点文档

## spi总线

maixpy支持其他板子, 经测试, 无法驱动本板子屏幕. 怀疑是接线有问题. 找来板子的接线图, 和各种硬件资料.

中景园1.14寸屏幕用的是TODO的芯片. 支持各种SPI通讯方式. 主要看屏幕控制芯片的手册.

SPI总线可以只靠一条线(3线接口), 或者控制/数据的电平信号线 + 真正的数据线(4线接口). 控制/数据的电平信号线为(高/低)的时候表示发送的这个字节是命令, 否则表示是数据(命令的参数). 另外, 这些传输都有时钟信号的同步.

片选信号变低, 选择芯片, 芯片准备接受数据. 不带(控制/数据的电平信号线)的时候传输9个bit, 第一个bit表示是控制还是数据. 带的时候连续传输8个bit,  传输第8个bit结束的时候对控制/数据信号进行采样, 判断是控制还是数据.

k210芯片带有fpio, 能够软件控制输出引脚和对应芯片内真正引脚的映射关系!! 可能类似fpga, 这个功能真是绝了, 可能是我见识太浅. 接spi线的时候, 一般spi接口的数据线还是直连, 其他复位, 控制/数据选择, 片选都可以直接在GPIO或者高速GPIO里面随便选一个. 因此不同板子要调整这些映射.

SPI总线也可以多条线, 4条,8条甚至16条等并行连接, 这个屏幕是典型的4线. 该板子可以外接的另外那个大屏幕似乎就是8数据线的spi.

4-line serial interface Ⅱ

|Pin Name | description|
|---|-----------------|
|CSX/CS| Chip selection signal|
|SCL| Clock signal|
|SDA| Serial data input data |
|WRX/RS| Data is regarded as a command when WRX is low. Data is regarded as a parameter or data when WRX is high|
|DCX| Clock signal |
|SDO| serial output data|

3 line interface Ⅰ spi 只有片选CSX, DCX时钟, SDA 输入输出
3 line interface Ⅱ 多了 SDO, 输入和输出引脚分开了.
4 line interface Ⅰ 只有片选CSX, WRX控制/数据(参数)选择, DCX时钟, SDA 输入输出 .
4 line interface Ⅱ同理.

DCX有时说是WRX一样的控制/数据选择. 可能是叫法的问题还是理解的问题???TODO

SCL --> LCD_WR
WRX/RS --> LCD_DC

LCD_WR是时钟
GPIOHS30在maixpy代表LCD复位
GPIOHS31代表控制/数据选择

```
fm.unregister(36, fm.fpioa.SPI0_SS3)
fm.register(37, fm.fpioa.SPI0_SS3)

fm.unregister(37, fm.fpioa.SPI0_SCLK)
fm.register(39, fm.fpioa.SPI0_SCLK)

fm.unregister(38, fm.fpioa.GPIOHS30)
fm.unregister(39, fm.fpioa.GPIOHS31)
fm.register(38, fm.fpioa.GPIOHS31)
```

LCD是片选3
```
+----------|-----------------------+
|   36     |     LCD_CS            |
+----------|-----------------------+
|   37     |     LCD_RST           |
+----------|-----------------------+
|   38     |     LCD_DC            |
+----------|-----------------------+
|   39     |     LCD_WR            |
+----------|-----------------------+
```
```
fm.unregister(36, fm.fpioa.SPI0_SS3)
fm.unregister(37, fm.fpioa.SPI0_SCLK)

fm.register(37, fm.fpioa.SPI0_SS3)
```

```
fpioa_set_function(37, FUNC_GPIOHS0 + RST_GPIONUM);
fpioa_set_function(38, FUNC_GPIOHS0 + DCX_GPIONUM);
fpioa_set_function(36, FUNC_SPI0_SS0+LCD_SPI_SLAVE_SELECT);
fpioa_set_function(39, FUNC_SPI0_SCLK);
```

SPI_FF_STANDARD： 标准
​ SPI_FF_DUAL： 双线
​ SPI_FF_QUAD： 四线
​ SPI_FF_OCTAL： 八线（SPI3 不支持）

## 屏幕控制芯片TODO

和芯片的交互基于SPI之后, 就是发送各种控制和数据. 芯片一般有nt35310和我现在的这个st7789. 首先是各种初始化的命令, 后面就是发送数据了. 显示的数据是一个个像素传送的, 有RGB565和SUV等颜色模式.

初始化首先发送SOFTWARE_RESET -(睡100ms 后面同理)-> SLEEP_OFF --> PIXEL_FORMAT_SET= 0x55 (表示是TODO模式) --> DISPALY_ON


## 屏幕定位

横着是x, 竖着是y方向

控制芯片的大小是`320x240`, 当设置屏幕大小是这个值的时候.
屏幕的大小是`240x135`, 刚好在中间. 左右空出40, 上下空出53/52左右
左上角大概在(40, 52), 右下角在(280, 187)左右. 发现代码中可以设置偏移, 把偏移设置成50 40(横屏模式), 和 40 52(竖屏模式) 就好了. 基本完美.
而且反色了... 和代码中写的颜色是反的, 代码中初始化是背景色红色, 白色的字. 不知道是不是有意为之, 或者有的屏幕就是这样. 反色之后的背景白里透蓝, 黑色的字.
那我就默认反色吧, 就和代码里的颜色一致了.

可以设置默认的屏幕大小. 按照保证允许软件传参更改的同时, 设置好能用的默认值的这个方针.


## TODO

micropython 解释器里面 摄像头能不能用

测试硅麦能不能用
找笑声检测模型

## 摄像头

引脚接的都是对的. 可以直接用, 但是不稳.
不知道为什么例程很稳, 而当前的maixpy经常会报错

TODO:
学习DVP接口, 学习摄像头相关的参数.
似乎是i2c? i2s? 2号 总线

## 麦克风

TODO:
完全没有声音... 全是0


