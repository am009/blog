---
title: 键盘之道-我的tmk之路
date: 2019/9/14 20:46:25
categories:
- 单片机
tags:
- TMK
- 键盘
---

# 键盘之道-我的tmk之路

真正属于自己的键盘！硬件上试图制作hasu usb to usb converter到Arduino pro mini + usb host shield mini 到最后购买成品。探索tmk（和qmk）固件，还有ahk（auto hot key）。
<!-- more -->
## 前言
我之所以会玩起键盘来，首先之前玩过Arduino，nodemcu，对单片机有了一些粗浅的了解。单片机能做的事情很多，让我迷恋的几个单片机应用的例子就包括硬件键盘记录器，以及自定义键盘的能力。
键盘，这么多按键，为输入而设计。而现在想要真正廉价的自定义按键阵列却又不容易！能不能让廉价的键盘上的所有键都能随心所欲地自定义，一键完成自己想要的任何功能吗？有的时候，确实存在一些操作，常用，但是却不能方便地完成，如果能直接按下桌子上的一个键就完成该有多好！
真正激起我的热情的是Linus Tech Tips的一期他们的视频剪辑师的视频（B站就有）他真正做到了自定义一整个键盘的功能，而且是非常方便而且功能强大的自定义（通过ahk，虽然不支持linux）！他作为视频剪辑师，天天接触ae或者pr，他就将一些常用功能绑定到了按键上，一键就能自动完成一些pr不支持设置快捷键的复杂设置，比如添加风声。这个操作需要各种选中，点击菜单，选择效果等等，但是利用编写好的ahk脚本就能一键完成！
这项技术强大的地方还不止于此，我没有用过hhkb，但是tmk固件真的有着不少强大的功能，而且不需要特别强的编程能力。在比较小的键盘上，无法装下所有键，难道真的没有办法完成全尺寸键盘的功能了吗？
有了tmk固件，就可以通过设置自定义的fn键设置一层新的键盘，当按下fn键，就可以把整个键盘转换为一层新的按键，让小尺寸键盘享受各种功能键（home， end，上下左右...），以及多媒体功能键。
## 原理

usb键盘的工作，涉及HID协议，了解不够的建议先补充一点相关知识。HID规定了所有的按键通过usb发送时的数据包，其中四个特殊的键，ctrl，alt，shift，winkey被设置成比特修饰位。
**也就是说，只要你是键盘设备，就不能够发送HID协议规定之外的键**，除非开发专属于自己设备的驱动。
usb转换器，是让任何usb键盘享受tmk功能的转换器。原以为像arduino这样的设备能够展控目光所及的所有电信号，但是实际上它还是跟不上usb的速率。于是它需要一个专攻usb通讯的小弟MAX3421芯片用来读取usb传输的HID协议，然后小弟在通过其他协议（好像叫boot协议）告诉arduino发生了什么。arduino pro micro或者arduino leonardo用的atmega32u4芯片由于支持发送usb的HID数据（可以模拟鼠标键盘），加上小弟就可以完美胜任读取键盘和发送数据给电脑的中间人的职能要求。这个芯片还是一些键盘的主控，所以也支持tmk固件，tmk固件也官方支持了usb2usb转换器。原生tmk固件的键盘价格昂贵，想让自己20几块的键盘或者自己喜欢的机械键盘能够用上tmk的强大自定义功能，就需要一个转换器（支持无线键盘，因为本质上还是HID设备。），（或者是teensy（相同芯片）做主控的键盘（没想到淘宝还发现了一个））。

## 最初的探索：自己的转换器
有三大主流方案，自己经过不懈地搜索，收集到了网上的资料。
资料来源：
1. geekhack.org “国外先进技术”
2. arduino的官方论坛
3. arduino的国内论坛
4. 百度搜索


有神人自己做的集成的u2u转换器的：

> 难道只能看着眼红的 HIDuino。虽然GitHub上有pcbdoc，但是没有BOM。。。
> https://www.arduino.cn/thread-23762-2-1.html

> 难不成要海淘的“国外成熟技术”，tmk交流论坛上的转换器
> https://geekhack.org/index.php?topic=69169.0
> 页面链接里就开源了pcb设计和BOM等，是kicad这个设计软件的
> https://github.com/tmk/USB2USB_Converter

有试图把小巧的arduino集成usb host shield mini的：

usb host shield mini 是3.7v供电，无论是自己的供电还是键盘的供电都成了问题（还是要用3.7v的arduino pro mini了）。。。而且还是有着一些兼容性的问题。不过听说teensy没有兼容性的问题？

> 在 Arduino Min (3.3V,8MHz) 上使用 Usb Host Shield Mini
> https://www.arduino.cn/thread-76102-1-1.html
> http://www.lab-z.com/cuhm/
> https://forum.arduino.cc/index.php?topic=325930.0
> https://www.pjrc.com/teensy/td_libs_USBHostShield.html
> https://forum.sparkfun.com/viewtopic.php?f=32&t=34873


> 国外兄弟的teensy成功案例
> https://www.arduino.cn/thread-81435-1-1.html


> 下面这位兄弟虽然做的是蓝牙转换器，但是也遇到了读取键盘的难题，他们却能够用好mini版的usb host shield。蓝牙版的转换器既然可编程，自然有定制键盘的潜能，虽然可能不能用上tmk固件。
> https://kevins.pro/arduino_keyboard
> https://github.com/KevinsBobo/arduino_keyboard

最简单的方案最后还是用成熟的usb host shield，就是体积太大了。
这个就不贴链接了，没有收藏。

> 修改 USB Host 库支持 Leonrado 模拟键盘鼠标
> https://www.arduino.cn/forum.php?mod=viewthread&tid=45194&highlight=usb%2Bhost

另外，偶然发现arduino due这个arm的单片机强大到可以读取usb信号，有成为转换器的潜力（虽然也没有tmk）

### 投靠成品
最后差点选择了diy hasu开源的硬件，连电路板都打出来了，最后还是选择了购买找到的成品。
咸鱼上虽然流传着一些nano版的转换器，但还是选择了一位键盘圈子里的“群友”的u2u转换器。


## ahk + 2nd keyboard ---Windows下的第二键盘的终极解决方案
### 原理
只能说ahk太强了，在ubuntu绝对找不到这么强的软件（虽然linux桌面环境本来就不是特别成熟）。Taran自定义的的固件在发送键前会按下f24，放开时也会松开f24，这操作可以被ahk检测到，从而识别第二个键盘的按键而不和原来的键盘混淆。还可以最多支持其他3把键盘的自定义（通过使用f24，f23和f22）
### 步骤
这个在Linus tech tips 的youtube视频下有各种链接，这里只摘录github链接。
这个只需要用到hasu_usb下的文件
https://github.com/TaranVH/2nd-keyboard

首先下载qmk_driver_installer的压缩包版，插上并使用install_all_drivers.bat(只能说没想到qmk和tmk在u2u上使用的是不同的驱动，不能用zadig装驱动)
然后就可以通过QMKtoolbox，安装这个github上已经有的bin固件了。
然后就可以使用ahk享受了！

之前还在担心是不是设备不兼容，我感觉现在国内所有的转换器，基本上都是和这个国外的Hasu u2u converter 兼容的，所以可以放心刷写，刷错了还可以刷回来，bootloader好像都是出厂的，所以不用担心。

## 探索tmk固件
拿到转换器，探索完2nd_keyboard，我又贪得无厌地（蛋疼地）思考，如果我要在ubuntu下使用怎么办？
发现ubuntu下还是有一些可以模拟鼠标键盘操作的命令行工具的，如果能够按一个键就启动对应的脚本我也就满足了
ubuntu 的gnome的设置里确实有设置快捷键的地方，可以执行一些命令，但是不支持四个修饰键之外的键修饰。
我想到，现在的键盘对HID协议规定的按键完全没有达到极限，许多按键完全没有被利用。比如一些其他语言的专用符号等。就算能把键盘上所有（大部分）键替换成不常用的键，电脑上拦截这些键那也足够了啊
转念一想，要是我能把四个修饰键（ctrl,alt,win,shift）都同时按下，应该不会和什么其他快捷键冲突，只要在第二个键盘上这样设置，不就能完美使用第二块键盘了吗？

直到我遇到了这篇文章，我才知道什么是tmk
> https://github.com/tmk/tmk_keyboard/blob/master/tmk_core/doc/keymap.md

tmk把键盘设计为层，让一些键成为切换层的按键（可以修饰也可以作为开关），这样达到了拓展键盘布局的能力。
而一些其他的特殊操作则通过fn键来实现。可以是宏（一系列按键操作），也可以是自带修饰键的按键（比如一个键代表Ctrl+C）

拥有全尺寸键盘的我原本不是很感兴趣这种层切换，但是没想到tmk固件不但支持多媒体键，还支持鼠标上的键，这让我非常激动，从此无论是调音量还是没有鼠标，都可以通过我的键盘完成！于是我现在的键盘布局是这样的：
第一层：大键盘基本不变，只是把自己从来没用过的右键菜单键改成计算器键。而小键盘就改成了鼠标操作区。右上角的三个没用的键（PrtScr,Scrool Lock,Pause Break）和小键盘锁一起改成带开关操作的四个修饰键，这样只要双击这四个键就可以绑定（大部分）键盘成四个修饰键全按的快捷键。

空格键单按是空格，修饰键时是切换层的fn
第二层：首先回归小键盘的功能，但是小键盘的加减改成了音量加减，旁边的\*改成了静音键。
上下左右键变成了鼠标滚轮的上下左右，便于翻阅网页等，wsad则变成了跳跃的上下，Home，End。右上角三个从来没用过的键变成了power，sleep，wake
左上角的~变成mycomputer，大键盘右上角的加减改成了调整亮度。

感谢TMK Keymap Generator，让我能方便地改键位而不用改源码编译

> 现在已经（基本上）不需要编译源码了
> https://www.cnblogs.com/zhuxiaoxi/p/10954703.html

键盘布局如下
```
["Esc",{x:1},"F1","F2","F3","F4",{x:0.5},"F5","F6","F7","F8",{x:0.5},"F9","F10","F11","F12",{x:0.25},"Fn5","Fn6","Fn7"],
[{y:0.5},"~\n`","!\n1","@\n2","#\n3","$\n4","%\n5","^\n6","&\n7","*\n8","(\n9",")\n0","_\n-","+\n=",{w:2},"Backspace",{x:0.25},"Insert","Home","PgUp",{x:0.25},"Fn8","/","*","-"],
[{w:1.5},"Tab","Q","W","E","R","T","Y","U","I","O","P","{\n[","}\n]",{w:1.5},"|\n\\",{x:0.25},"Delete","End","PgDn",{x:0.25},"btn1","mouseup","btn2",{h:2},"accel2"],
[{w:1.75},"Caps Lock","A","S","D","F","G","H","J","K","L",":\n;","\"\n'",{w:2.25},"Enter",{x:3.5},"mouseleft","mousedown","mouseright"],
[{w:2.25},"Shift","Z","X","C","V","B","N","M","<\n,",">\n.","?\n/",{w:2.75},"RShift",{x:1.25},"↑",{x:1.25},"btn4","btn3","btn5",{h:2},"PEnter"],
[{w:1.25},"Ctrl",{w:1.25},"Win",{w:1.25},"Alt",{w:6.25},"Fn0",{w:1.25},"RAlt",{w:1.25},"RWin",{w:1.25},"calc",{w:1.25},"RCtrl",{x:0.25},"←","↓","→",{x:0.25,w:2},"0\nIns",".\nDel"]

["Esc",{x:1},"F1","F2","F3","F4",{x:0.5},"F5","F6","F7","F8",{x:0.5},"F9","F10","F11","F12",{x:0.25},"power","sleep","wake"],
[{y:0.5},"mycomp","!\n1","@\n2","#\n3","$\n4","%\n5","^\n6","&\n7","*\n8","(\n9",")\n0","sbdn","sbup",{w:2},"Backspace",{x:0.25},"Insert","Home","PgUp",{x:0.25},"Num Lock","/","mute","voldown"],
[{w:1.5},"Tab","btn1","up","E","R","T","Y","U","I","O","P","{\n[","}\n]",{w:1.5},"|\n\\",{x:0.25},"Delete","End","PgDn",{x:0.25},"7\nHome","8\n↑","9\nPgUp",{h:2},"volup"],
[{w:1.75},"Caps Lock","Home","down","End","find","G","H","J","K","L",":\n;","\"\n'",{w:2.25},"Enter",{x:3.5},"4\n←","5","6\n→"],
[{w:2.25},"Shift","undo","cut","copy","paste","again","N","M","<\n,",">\n.","?\n/",{w:2.75},"RShift",{x:1.25},"wheelup",{x:1.25},"1\nEnd","2\n↓","3\nPgDn",{h:2},"PEnter"],
[{w:1.25},"Ctrl",{w:1.25},"Win",{w:1.25},"Alt",{w:6.25},"Fn0",{w:1.25},"RAlt",{w:1.25},"RWin",{w:1.25},"calc",{w:1.25},"RCtrl",{x:0.25},"wheelleft","wheeldown","wheelright",{x:0.25,w:2},"0\nIns",".\nDel"]
```

## 感叹
探索冷门技术还是要和国际接轨，英文论坛上往往有着最前沿的（成熟的）消息或者成果，要学好英语啊。
另外似乎不需要考虑bootloader的问题，好像tmk固件可以通过atmega32u4出厂的bootloader通过usb烧写。arduino的bootloader就不知道了。

http://mouse.zol.com.cn/677/6774443.html

tmk是什么？tmk是一款开源的键盘固件。


