---
title: 学习使用pwndbg
date: 2019/7/8 20:46:25
categories:
- CTF
tags:
- CTF
- pwndbg
- pwngdb
- peda
---

# 学习使用pwngdb

关键词：pwngdb， peda
在linux下，不使用ida，就没有特别好的调试器。
有模仿ollydbg的edb，再有就是改进版的gdb了，peda就是为了二进制逆向设计的gdb插件。
<!-- more -->
平常使用gdb一般是在有源码的情况下，gcc编译时加上调试选项，就会在源代码里面嵌入源代码和调试用的符号表，供gdb调试。
pwngdb，加了插件的gdb，需要熟练掌握。

### 安装
https://blog.csdn.net/xiangshangbashaonian/article/details/80463882
最好先安装pwndbg再装peda
主要是配置文件~/.gdbinit中pwndbg的加载文件要放在peda之前

相关命令：
https://blog.csdn.net/qq_38783875/article/details/80382294

有关peda，看github就够了，它列出了核心命令的用法
https://github.com/longld/peda
pwndbg的文档
https://github.com/pwndbg/pwndbg/blob/dev/FEATURES.md

pwndbg和ida可以集成，使用根目录下的脚本，难道是在idapython里面直接运行？应该是那个ida上方菜单出直接加载。

### 常用命令总结

查看寄存器的值 ：p $寄存器

修改寄存器的值：set var $寄存器 = expr

给存储在address地址的变量类型为type的变量赋值： set {type}address=expr

#### 查看内存的值
x/<n/f/u> <addr>
1） n 是一个正整数，表示显示内存的长度，也就是说从当前地址向后显示几个地址的内容
    
2） f 表示显示的格式
参数 f 的可选值：

x 按十六进制格式显示变量。
d 按十进制格式显示变量。
u 按十六进制格式显示无符号整型。
o 按八进制格式显示变量。
t 按二进制格式显示变量。
a 按十六进制格式显示变量。
c 按字符格式显示变量。
f 按浮点数格式显示变量。
    
3） u 表示将多少个字节作为一个值取出来，如果不指定的话，GDB默认是4个bytes，如果不指定的话，默认是4个bytes。当我们指定了字节长度后，GDB会从指内存定的内存地址开始，读写指定字节，并把其当作一个值取出来。

参数 u 可以用下面的字符来代替：

b 表示单字节

h 表示双字节

w 表示四字 节

g 表示八字节

#### 查看字符串
p (char*)0x23b744a98
p 表示print
下面的命令可以修改显示的最大长度
(gdb) set print elements 0 #unlimited
(gdb) show print elements

#### pwngdb的magic
这个命令可以显示出各种重要的数据。
各种system类的函数地址，gadgets地址，和hook地址。
这个神奇的命令来自pwngdb，发现自己没装
https://github.com/scwuaptx/Pwngdb
另外还有只看到ctfwiki里有的gef：
https://github.com/hugsy/gef

#### 其他技巧
使用watch 可以下内存断点
可以使用libc里面的符号。比如environ这个符号存的就是环境变量的位置

### 自动化控制
使用pygdbmi这个库控制gdb。为了能够集成pwntools，需要了解一下pwntools的attach原理。经过考察，既然gdbgui(基于浏览器的gdb图形化界面)都使用了它，这个库还是不错的。
log_level设置为debug之后，就可以看到gdb.attach这个函数其实就是调用了gdb的-q参数载入elf，加上pid参数让gdb attach，加上一个临时文件作为启动时的脚本。
```
[DEBUG] Launching a new terminal: ['/usr/bin/mate-terminal', '-x', 'sh', '-c', '/usr/bin/gdb -q  "/mnt/hgfs/D/ctf/pwn/ret2csu/\xe4\xb8\x80\xe6\xad\xa5\xe4\xb8\x80\xe6\xad\xa5level5/level5" 2309 -x "/tmp/pwnUKgUOy.gdb"']
```
也就是说
https://github.com/cs01/pygdbmi
https://cs01.github.io/pygdbmi/#programmatic-control-over-gdb
```
GdbController

__init__(self, gdb_path='gdb', gdb_args=None, time_to_check_for_additional_output_sec=0.2, rr=False, verbose=False)
```
https://stackoverflow.com/questions/3482869/invoke-and-control-gdb-from-python

只要
```python
from pygdbmi.gdbcontroller import GdbController
if __name__ == '__main__':
    gdbmi = GdbController(gdb_args=['-q', '/path/to/elf', '<pid>'])
    response = gdbmi.write('b main')
    response = gdbmi.write('target remote localhost:2331')
    response = gdbmi.write('mon reset 0')
    response = gdbmi.write('c')
```
但是就不会弹出来窗口了。。。它是把gdb作为一个子进程
所以想要能够一边有一个单独的gdb窗口在边上，能够手动交互，一边又能够在右边命令行用命令进行交互是很不容易的。
所以还是老实地使用gdb启动脚本吧。学学gdb函数定义也是不错的
或者能够使用python和自己的terminal交互，模拟手动输入


### 使用实例
以pwnable.kr的bof为例

首先使用file命令加载进去
使用r表示运行
r是重新开始运行，会中止现在的进程开启新的进程
c表示continue才是继续运行
```
gdb-peda$ file bof
Reading symbols from bof...(no debugging symbols found)...done.
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
gdb-peda$ r
Starting program: /home/wjk/bof
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
overflow me :
Nag^Hh..
Nah..
[Inferior 1 (process 1181) exited normally]
Warning: not running
gdb-peda$
```
观察结果发现，直接运行没有设置断点，就会正常运行

接下来，使用 b main表示在main出下断点
还有断点神器(pie)
> b *$rebase(0x100)

再次运行
```
gdb-peda$ b
No default breakpoint address now.
gdb-peda$ b main
Breakpoint 1 at 0x5655568d
gdb-peda$ r
Starting program: /home/wjk/bof
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
[----------------------------------registers-----------------------------------]
EAX: 0xf7fc1dbc --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
EBX: 0x0
ECX: 0x70a8dc7e
EDX: 0xffffd654 --> 0x0
ESI: 0xf7fc0000 --> 0x1b1db0
EDI: 0xf7fc0000 --> 0x1b1db0
EBP: 0xffffd628 --> 0x0
ESP: 0xffffd628 --> 0x0
EIP: 0x5655568d (<main+3>:      and    esp,0xfffffff0)
EFLAGS: 0x296 (carry PARITY ADJUST zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x56555689 <func+93>:        ret
   0x5655568a <main>:   push   ebp
   0x5655568b <main+1>: mov    ebp,esp
=> 0x5655568d <main+3>: and    esp,0xfffffff0
   0x56555690 <main+6>: sub    esp,0x10
   0x56555693 <main+9>: mov    DWORD PTR [esp],0xdeadbeef
   0x5655569a <main+16>:        call   0x5655562c <func>
   0x5655569f <main+21>:        mov    eax,0x0
[------------------------------------stack-------------------------------------]
0000| 0xffffd628 --> 0x0
0004| 0xffffd62c --> 0xf7e26637 (<__libc_start_main+247>:       add    esp,0x10)
0008| 0xffffd630 --> 0x1
0012| 0xffffd634 --> 0xffffd6c4 --> 0xffffd7f5 ("/home/wjk/bof")
0016| 0xffffd638 --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
0020| 0xffffd63c --> 0x0
0024| 0xffffd640 --> 0x0
0028| 0xffffd644 --> 0x0
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x5655568d in main ()
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
gdb-peda$
```
可以看到，如果在断点停下，就会像普通的dbg一样，显示出当前的1.寄存器2.反汇编指令3.栈

接下来使用
step/s          单步步入
next/n          单步步过
si --- 真正的单步
ni --- 真正的下一步
有时候可能因为cpu的流水线问题，一次单步会执行多个指令，所以需要加上i
```
gdb-peda$
gdb-peda$
gdb-peda$ next
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
[----------------------------------registers-----------------------------------]
EAX: 0xf7fc1dbc --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
EBX: 0x0
ECX: 0x70a8dc7e
EDX: 0xffffd654 --> 0x0
ESI: 0xf7fc0000 --> 0x1b1db0
EDI: 0xf7fc0000 --> 0x1b1db0
EBP: 0xffffd628 --> 0x0
ESP: 0xffffd620 --> 0xf7fc0000 --> 0x1b1db0
EIP: 0x56555690 (<main+6>:      sub    esp,0x10)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x5655568a <main>:   push   ebp
   0x5655568b <main+1>: mov    ebp,esp
   0x5655568d <main+3>: and    esp,0xfffffff0
=> 0x56555690 <main+6>: sub    esp,0x10
   0x56555693 <main+9>: mov    DWORD PTR [esp],0xdeadbeef
   0x5655569a <main+16>:        call   0x5655562c <func>
   0x5655569f <main+21>:        mov    eax,0x0
   0x565556a4 <main+26>:        leave
[------------------------------------stack-------------------------------------]
0000| 0xffffd620 --> 0xf7fc0000 --> 0x1b1db0
0004| 0xffffd624 --> 0xf7fc0000 --> 0x1b1db0
0008| 0xffffd628 --> 0x0
0012| 0xffffd62c --> 0xf7e26637 (<__libc_start_main+247>:       add    esp,0x10)
0016| 0xffffd630 --> 0x1
0020| 0xffffd634 --> 0xffffd6c4 --> 0xffffd7f5 ("/home/wjk/bof")
0024| 0xffffd638 --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
0028| 0xffffd63c --> 0x0
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
0x56555690 in main ()
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
gdb-peda$
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':

Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
[----------------------------------registers-----------------------------------]
EAX: 0xf7fc1dbc --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
EBX: 0x0
ECX: 0x70a8dc7e
EDX: 0xffffd654 --> 0x0
ESI: 0xf7fc0000 --> 0x1b1db0
EDI: 0xf7fc0000 --> 0x1b1db0
EBP: 0xffffd628 --> 0x0
ESP: 0xffffd610 --> 0xf7fc03dc --> 0xf7fc11e0 --> 0x0
EIP: 0x56555693 (<main+9>:      mov    DWORD PTR [esp],0xdeadbeef)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x5655568b <main+1>: mov    ebp,esp
   0x5655568d <main+3>: and    esp,0xfffffff0
   0x56555690 <main+6>: sub    esp,0x10
=> 0x56555693 <main+9>: mov    DWORD PTR [esp],0xdeadbeef
   0x5655569a <main+16>:        call   0x5655562c <func>
   0x5655569f <main+21>:        mov    eax,0x0
   0x565556a4 <main+26>:        leave
   0x565556a5 <main+27>:        ret
[------------------------------------stack-------------------------------------]
0000| 0xffffd610 --> 0xf7fc03dc --> 0xf7fc11e0 --> 0x0
0004| 0xffffd614 ("PRUV\271VUV")
0008| 0xffffd618 --> 0x565556b9 (<__libc_csu_init+9>:   add    ebx,0x193b)
0012| 0xffffd61c --> 0x0
0016| 0xffffd620 --> 0xf7fc0000 --> 0x1b1db0
0020| 0xffffd624 --> 0xf7fc0000 --> 0x1b1db0
0024| 0xffffd628 --> 0x0
0028| 0xffffd62c --> 0xf7e26637 (<__libc_start_main+247>:       add    esp,0x10)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
0x56555693 in main ()
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
gdb-peda$
```
可见只要使用过一次next之后，按回车会重复之前的next
现在我们到下面的call处，使用插件提供的查看函数参数的命令
dumpargs命令 Display arguments passed to a function when stopped at a call instruction
```
gdb-peda$

Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
[----------------------------------registers-----------------------------------]
EAX: 0xf7fc1dbc --> 0xffffd6cc --> 0xffffd803 ("XDG_SESSION_ID=2")
EBX: 0x0
ECX: 0x70a8dc7e
EDX: 0xffffd654 --> 0x0
ESI: 0xf7fc0000 --> 0x1b1db0
EDI: 0xf7fc0000 --> 0x1b1db0
EBP: 0xffffd628 --> 0x0
ESP: 0xffffd610 --> 0xdeadbeef
EIP: 0x5655569a (<main+16>:     call   0x5655562c <func>)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x5655568d <main+3>: and    esp,0xfffffff0
   0x56555690 <main+6>: sub    esp,0x10
   0x56555693 <main+9>: mov    DWORD PTR [esp],0xdeadbeef
=> 0x5655569a <main+16>:        call   0x5655562c <func>
   0x5655569f <main+21>:        mov    eax,0x0
   0x565556a4 <main+26>:        leave
   0x565556a5 <main+27>:        ret
   0x565556a6:  nop
Guessed arguments:
arg[0]: 0xdeadbeef
[------------------------------------stack-------------------------------------]
0000| 0xffffd610 --> 0xdeadbeef
0004| 0xffffd614 ("PRUV\271VUV")
0008| 0xffffd618 --> 0x565556b9 (<__libc_csu_init+9>:   add    ebx,0x193b)
0012| 0xffffd61c --> 0x0
0016| 0xffffd620 --> 0xf7fc0000 --> 0x1b1db0
0020| 0xffffd624 --> 0xf7fc0000 --> 0x1b1db0
0024| 0xffffd628 --> 0x0
0028| 0xffffd62c --> 0xf7e26637 (<__libc_start_main+247>:       add    esp,0x10)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
0x5655569a in main ()
Python Exception <class 'AttributeError'> module 'pwndbg' has no attribute 'commands':
gdb-peda$ ddumpargs
Guessed arguments:
arg[0]: 0xdeadbeef
gdb-peda$
```
backtrace/bt：查看函数调用信息(堆栈)
根据ebp的指针跳进去，看到递归调用的信息，和打印错误时一样。
```
gdb-peda$ info stack
#0  0x5655569a in main ()
#1  0xf7e26637 in __libc_start_main (main=0x5655568a <main>, argc=0x1, argv=0xffffd6c4,
    init=0x565556b0 <__libc_csu_init>, fini=0x56555720 <__libc_csu_fini>,
    rtld_fini=0xf7fe8880 <_dl_fini>, stack_end=0xffffd6bc) at ../csu/libc-start.c:291
#2  0x56555561 in _start ()
```
另外如果想要重启当然是使用kill和run









vmmap命令可以查看地址映射信息
```
gdb-peda$ vmmap
Start      End        Perm      Name
0x56555000 0x56556000 r-xp      /home/wjk/bof
0x56556000 0x56557000 r--p      /home/wjk/bof
0x56557000 0x56558000 rw-p      /home/wjk/bof
0xf7e0d000 0xf7e0e000 rw-p      mapped
0xf7e0e000 0xf7fbe000 r-xp      /lib/i386-linux-gnu/libc-2.23.so
0xf7fbe000 0xf7fc0000 r--p      /lib/i386-linux-gnu/libc-2.23.so
0xf7fc0000 0xf7fc1000 rw-p      /lib/i386-linux-gnu/libc-2.23.so
0xf7fc1000 0xf7fc4000 rw-p      mapped
0xf7fd4000 0xf7fd5000 rw-p      mapped
0xf7fd5000 0xf7fd8000 r--p      [vvar]
0xf7fd8000 0xf7fd9000 r-xp      [vdso]
0xf7fd9000 0xf7ffc000 r-xp      /lib/i386-linux-gnu/ld-2.23.so
0xf7ffc000 0xf7ffd000 r--p      /lib/i386-linux-gnu/ld-2.23.so
0xf7ffd000 0xf7ffe000 rw-p      /lib/i386-linux-gnu/ld-2.23.so
0xfffdd000 0xffffe000 rw-p      [stack]
```


readelf和elfheader可以查看段信息
```
gdb-peda$ elfheader
.interp = 0x56555154
.note.ABI-tag = 0x56555168
.note.gnu.build-id = 0x56555188
.gnu.hash = 0x565551ac
.dynsym = 0x565551e0
.dynstr = 0x565552c0
.gnu.version = 0x56555378
.gnu.version_r = 0x56555394
.rel.dyn = 0x565553d4
.rel.plt = 0x5655543c
.init = 0x56555474
.plt = 0x565554b0
.text = 0x56555530
.fini = 0x56555768
.rodata = 0x56555784
.eh_frame_hdr = 0x565557ac
.eh_frame = 0x565557e0
.ctors = 0x56556f00
.dtors = 0x56556f08
.jcr = 0x56556f10
.dynamic = 0x56556f14
.got = 0x56556fe4
.got.plt = 0x56556ff4
.data = 0x5655701c
.bss = 0x56557024
gdb-peda$ readelf
.interp = 0x56555154
.note.ABI-tag = 0x56555168
.note.gnu.build-id = 0x56555188
.gnu.hash = 0x565551ac
.dynsym = 0x565551e0
.dynstr = 0x565552c0
.gnu.version = 0x56555378
.gnu.version_r = 0x56555394
.rel.dyn = 0x565553d4
.rel.plt = 0x5655543c
.init = 0x56555474
.plt = 0x565554b0
.text = 0x56555530
.fini = 0x56555768
.rodata = 0x56555784
.eh_frame_hdr = 0x565557ac
.eh_frame = 0x565557e0
.ctors = 0x56556f00
.dtors = 0x56556f08
.jcr = 0x56556f10
.dynamic = 0x56556f14
.got = 0x56556fe4
.got.plt = 0x56556ff4
.data = 0x5655701c
.bss = 0x56557024
gdb-peda$
```


