---
title: ucore lab1
date: 2019/9/29 13:46:25
categories:
- OS
tags:
- ucore
---

# ucore lab1
上来先看看Intel 80386 Programmer's Reference Manual, 1987 (HTML)吧。
http://www.logix.cz/michal/doc/i386/
<!-- more -->


## makefile
> https://chyyuu.gitbooks.io/ucore_os_docs/content/lab0/lab0_ref_ucore-resource.html


跟我一起写makefile
> https://seisman.github.io/how-to-write-makefile/functions.html
> 
makefile这鬼东西真tm功能强，强得我什么都看不懂。原来在那个眼花缭乱的makefile里面全是各种各样的函数。。。。
但是实际上不需要全部读懂，稍微看看就好，这个不是重点，而且好像这个脚本比较通用。

> https://blog.csdn.net/u013484370/article/details/50638353
```
# add files to packet: (#files, cc[, flags, packet, dir])
#此模板，就是真正在makefile中用来编译所有的目表文件，并生成makefile规则的模板。
define do_add_files_to_packet
#__temp_packet__用来记录所有的临时目标文件。
__temp_packet__ := $(call packetname,$(4))
ifeq ($$(origin $$(__temp_packet__)),undefined)
$$(__temp_packet__) :=
endif
__temp_objs__ := $(call toobj,$(1),$(5))
$$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(5))))
$$(__temp_packet__) += $$(__temp_objs__)
endef
```

其他资料：
> How to make an Operating System
> https://samypesse.gitbook.io/how-to-create-an-operating-system/chapter-3
> 别人的lab1实验报告:
> https://www.jianshu.com/p/2f95d38afa1d
> https://www.cnblogs.com/maruixin/p/3175894.html
> 

## 硬盘读取
> 《读取磁盘：LBA方式》
> https://www.cnblogs.com/mlzrq/p/10223060.html

这个写的比那个gitbook实验指导书讲得好一点
> LBA简介
> 磁盘读取发展
> 
> IO操作读取硬盘的三种方式：
> 
> chs方式 ：小于8G (8064MB)
> 
> LBA28方式：小于137GB
> 
> LBA48方式：小于144,000,000 GB
> 
> LBA方式访问使用了data寄存器，LBA寄存器（总共3个），device寄存器，command寄存器来完成的。
> 
> LBA28和LBA48方式：
> LBA28方式使用28位来描述一个扇区地址，最大支持128GB的硬磁盘容量。
> 
> LBA28的寄存器
> 
> 


|寄存器|	端口|	作用|
|-----|------|-----|
| data寄存器|	0x1F0	已经读取或写入的数据，大小为两个字节（16位数据)|每次读取1个word,反复循环，直到读完所有数据
|features寄存器|0x1F1|读取时的错误信息，写入时的额外参数
|sector count寄存器	|0x1F2|	指定读取或写入的扇区数
|LBA low寄存器	|0x1F3|	lba地址的低8位|
|LBA mid寄存器	|0x1F4|	lba地址的中8位|
|LBA high寄存器|	0x1F5|	lba地址的高8位|
|device寄存器	|0x1F6|	lba地址的前4位（占用device寄存器的低4位）<br />主盘值为0（占用device寄存器的第5位）<br />第6位值为1<br />LBA模式为1，CHS模式为0（占用device寄存器的第7位）<br />第8位值为1|
|command寄存器	|0x1F7|	读取，写入的命令，返回磁盘状态。<br />1 读取扇区:0x20 写入扇区:0x30<br />磁盘识别:0xEC
IDE通道1，读写0x1f0-0x1f7号端口
IDE通道2，读写0x170-0x17f号端口

---
Program header描述的是一个段在文件中的位置、大小以及它被放进内存后所在的位置和大小。
所以bootmain中的读取elf文件，需要把每一个段都按照指定的虚拟地址和size加载好。
```
readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```
但是为什么是ph->p_va & 0xFFFFFF？
去掉了前面一个字节。
答：应该不是因为硬盘的问题，可能是内存大小的限制
> https://blog.csdn.net/u012418573/article/details/73823524
> 2、链接地址VS加载地址
> （1）链接地址：是虚拟地址，代码中的绝对跳转地址和全局变量的地址都依赖于链接地址，链接地址改变时，这些地址也会改变，但相对跳转不依赖与链接地址。
> （2）加载地址：程序被加载到的物理地址
> （3）关系：链接地址经过地址转换要等于物理地址（加载地址）
> （4）内核的加载地址：0x100000处，参加bootmain.c
> （5）内核的链接地址：0xf0100000处，但是我们没有那么大的内存，故：ELFHDR->e_entry&0xFFFFFF

另外，这个问题好像不简单啊：
> ucore Lab2 调试时断点无效分析
> http://blog.sina.com.cn/s/blog_3dce1e7b0102x6t3.html

就是说，lab2的bootloader加载完成之后，会加载一个新的gdt段表，让地址多了一个KERNBASE(0xC0000000)。而lab1没有这个功能，就直接通过这样与一下来解决。


## 实模式到保护模式的切换

问：16位到32位的切换是瞬间完成的吗？

问：发现进入实模式之后的ljmp之后，cs自动变成8
答：代码段寄存器（CS）的内容不能由装载指令（如MOV）直接设置，而只能被那些会改变程序执行顺序的指令（如JMP、INT、CALL）间接地设置。
8代表序号为1的段描述符表。进入实模式之前就要设置好段描述符表。
> 下面摘自《计算机启动流程分析--以JOS为例（从BIOS到刚进入boot loader）》
> https://blog.csdn.net/old_memory/article/details/79572498

$CR0_PE_ON是CR0设置实模式或保护模式的开关，这里打开，表明接下来的地址都是32位的虚拟地址（必须注意，这里由于是刚开始，所有的虚拟地址和物理地址等价），然而，系统是怎样真正进入32位寻址的呢？以下代码：
```
ljmp    $PROT_MODE_CSEG, $protcseg
 
  .code32                     # Assemble for 32-bit mode
protcseg:
```
它的作用仅仅是跳转到下一行，但是ljmp有副作用：$PROT_MODE_CSEG的值会被加载，boot.S文件开头的宏表明：
```
.set PROT_MODE_CSEG, 0x8         # kernel code segment selector
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector
.set CR0_PE_ON,      0x1         # protected mode enable flag
```
它的值是0x8，这个值被存入CS寄存器，它会与GDT一起影响地址翻译。

摘录结束，另外在文件bootblock.asm中有
```
    ljmp $PROT_MODE_CSEG, $protcseg
    7c2d:	ea                   	.byte 0xea
    7c2e:	32 7c 08 00          	xor    0x0(%eax,%ecx,1),%bh
```
这说明反汇编失败了？所以ljmp实际上是32位才有的指令吧。
可是为什么在gdb里面却显示
0x7c2d <seta20+25>:	ljmp   $0xb866,$0x87c32
只是跳转到7c32，为什么有这么多一堆东西？
严重怀疑在jmp之前还是十六位的代码。

另外这篇文章说得特别好啊，说出了内幕：
> https://blog.csdn.net/dog250/article/details/5303304
> ljmp的含义是长跳，长跳主要就是重新加载寄存器，32位保护模式主要体现在段寄存器，具有可以参考段选择子和段描述符的概念，如果不用长跳的话，那么段寄存器不会重新加载，后面的取指结果仍然是老段寄存器中的值，当然保护模式不会生效了，Intel手册上有讲可见寄存器和不可见寄存器的篇章，可以看一下，其实实模式就是保护模式的一种权限全开放的特殊情况，就是说段寄存器左移相当于右边添加0，而这添加的0可以看做保护模式的RPL，RPL为0代表Intel的0环，当然是全权限了。
> 
> 不过Intel的实模式的概念实属不得已而为之，现在的意义已经不大了，从实模式启动然后跳转到保护模式纯粹是在绕圈子，没有实质的意义，商业上为了保护以前的投资不得不将技术做的没有意义的复杂...



---
> 【补充】保护模式下，有两个段表：GDT（Global Descriptor Table）和LDT（Local Descriptor Table），每一张段表可以包含8192 (2^13)个描述符[1]，因而最多可以同时存在2 * 2^13 = 2^14 个段。虽然保护模式下可以有这么多段，逻辑地址空间看起来很大，但实际上段并不能扩展物理地址空间，很大程度上各个段的地址空间是相互重叠的。目前所谓的64TB（2^(14+32) =2^46 ）逻辑地址空间是一个理论值，没有实际意义。在32位保护模式下，真正的物理空间仍然只有2^32字节那么大。注：在ucore lab中只用到了GDT，没有用LDT。

没想到说32位系统最多有4gb内存不是假的啊。。。换页还能换着用超出4gb的内存吗？

另外为什么是2^13次方？
因为段寄存器占用了三个比特位用来干别的。参见
https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_3_2_1_protection_mode.html
这实验指导书这次摘录得特别好啊！

---

## 段描述符

问： 如果某个段描述符要求的权限级别是3，那么我通过加载段寄存器，index选择这个描述符，但是权限级是0，那这样不就让当前权限变成0了？
答：对应的段因为权限是三，本来就没有对权限做限制。所以访问起来并没有什么区别。
但是如果再切换的话就会造成影响，使得能够切换到ring0的数据段。
难道是真正切换时，权限级不由自己加载的段寄存器的值确定，而是由选择出来的段描述符的权限级确定？

特权级和特权级转移
https://www.jianshu.com/p/377f473dd0a9


这段描述符的结构很重要。
![](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1_figs/image003.png)
* 段基地址：规定线性地址空间中段的起始地址。在80386保护模式下，段基地址长32位。因为基地址长度与寻址地址的长度相同，所以任何一个段都可以从32位线性地址空间中的任何一个字节开始，而不象实方式下规定的边界必须被16整除。
* 段界限：规定段的大小。在80386保护模式下，段界限用20位表示，而且段界限可以是以字节为单位或以4K字节为单位。
* 段属性：确定段的各种性质。:
    * 段属性中的粒度位（Granularity），用符号G标记。G=0表示段界限以字节位位单位，20位的界限可表示的范围是1字节至1M字节，增量为1字节；G=1表示段界限以4K字节为单位，于是20位的界限可表示的范围是4K字节至4G字节，增量为4K字节。
    * 类型（TYPE）：用于区别不同类型的描述符。可表示所描述的段是代码段还是数据段，所描述的段是否可读/写/执行，段的扩展方向等。
    * 描述符特权级（Descriptor Privilege Level）（DPL）：用来实现保护机制。
    * 段存在位（Segment-Present bit）：如果这一位为0，则此描述符为非法的，不能被用来实现地址转换。如果一个非法描述符被加载进一个段寄存器，处理器会立即产生异常。图5-4显示了当存在位为0时，描述符的格式。操作系统可以任意的使用被标识为可用（AVAILABLE）的位。
    * 已访问位（Accessed bit）：当处理器访问该段（当一个指向该段描述符的选择子被加载进一个段寄存器）时，将自动设置访问位。操作系统可清除该位。

http://www.logix.cz/michal/doc/i386/chp06-03.htm#06-03-01
```
Figure 6-1. Protection Fields of Segment Descriptors
                           DATA SEGMENT DESCRIPTOR

  31                23                15                7               0
 +-----------------+-+-+-+-+---------+-+-----+---------+-----------------+
 |#################|#|#|#|A| LIMIT   |#|     |  TYPE   |#################|
 |###BASE 31..24###|G|B|0|V| 19..16  |P| DPL |         |###BASE 23..16###| 4
 |#################|#|#|#|L|         |#|     |1|0|E|W|A|#################|
 |-----------------+-+-+-+-+---------+-+-----+-+-+-+-+-+-----------------|
 |###################################|                                   |
 |########SEGMENT BASE 15..0#########|        SEGMENT LIMIT 15..0        | 0
 |###################################|                                   |
 +-----------------+-----------------+-----------------+-----------------+

                        EXECUTABLE SEGMENT DESCRIPTOR

  31                23                15                7               0
 +-----------------+-+-+-+-+---------+-+-----+---------+-----------------+
 |#################|#|#|#|A| LIMIT   |#|     |  TYPE   |#################|
 |###BASE 31..24###|G|D|0|V| 19..16  |P| DPL |         |###BASE 23..16###| 4
 |#################|#|#|#|L|         |#|     |1|0|C|R|A|#################|
 |-----------------+-+-+-+-+---------+-+-----+-+-+-+-+-+-----------------|
 |###################################|                                   |
 |########SEGMENT BASE 15..0#########|        SEGMENT LIMIT 15..0        | 0
 |###################################|                                   |
 +-----------------+-----------------+-----------------+-----------------+

                         SYSTEM SEGMENT DESCRIPTOR

  31                23                15                7               0
 +-----------------+-+-+-+-+---------+-+-----+-+-------+-----------------+
 |#################|#|#|#|A| LIMIT   |#|     | |       |#################|
 |###BASE 31..24###|G|X|0|V| 19..16  |P| DPL |0| TYPE  |###BASE 23..16###| 4
 |#################|#|#|#|L|         |#|     | |       |#################|
 |-----------------+-+-+-+-+---------+-+-----+-+-------+-----------------|
 |###################################|                                   |
 |########SEGMENT BASE 15..0#########|       SEGMENT LIMIT 15..0         | 0
 |###################################|                                   |
 +-----------------+-----------------+-----------------+-----------------+
        A   - ACCESSED                              E   - EXPAND-DOWN
        AVL - AVAILABLE FOR PROGRAMMERS USE         G   - GRANULARITY
        B   - BIG                                   P   - SEGMENT PRESENT
        C   - CONFORMING                            R   - READABLE
        D   - DEFAULT                               W   - WRITABLE
        DPL - DESCRIPTOR PRIVILEGE LEVEL

Table 6-1. System and Gate Descriptor Types
Code      Type of Segment or Gate

  0       -reserved
  1       Available 286 TSS
  2       LDT
  3       Busy 286 TSS
  4       Call Gate
  5       Task Gate
  6       286 Interrupt Gate
  7       286 Trap Gate
  8       -reserved
  9       Available 386 TSS
  A       -reserved
  B       Busy 386 TSS
  C       386 Call Gate
  D       -reserved
  E       386 Interrupt Gate
  F       386 Trap Gate
```



## IDT的结构
```
Figure 9-1. IDT Register and Table
                                              INTERRUPT DESCRIPTOR TABLE
                                              +------+-----+-----+------+
                                        +---->|      |     |     |      |
                                        |     |- GATE FOR INTERRUPT #N -|
                                        |     |      |     |     |      |
                                        |     +------+-----+-----+------+
                                        |     *                         *
                                        |     *                         *
                                        |     *                         *
                                        |     +------+-----+-----+------+
                                        |     |      |     |     |      |
                                        |     |- GATE FOR INTERRUPT #2 -|
                                        |     |      |     |     |      |
                                        |     |------+-----+-----+------|
            IDT REGISTER                |     |      |     |     |      |
                                        |     |- GATE FOR INTERRUPT #1 -|
                    15            0     |     |      |     |     |      |
                   +---------------+    |     |------+-----+-----+------|
                   |   IDT LIMIT   |----+     |      |     |     |      |
  +----------------+---------------|          |- GATE FOR INTERRUPT #0 -|
  |            IDT BASE            |--------->|      |     |     |      |
  +--------------------------------+          +------+-----+-----+------+
   31                             0
```
这里的idt limit是idt的长度减1

```
Figure 9-3. 80306 IDT Gate Descriptors
                                80386 TASK GATE
   31                23                15                7                0
  +-----------------+-----------------+---+---+---------+-----------------+
  |#############(NOT USED)############| P |DPL|0 0 1 0 1|###(NOT USED)####|4
  |-----------------------------------+---+---+---------+-----------------|
  |             SELECTOR              |#############(NOT USED)############|0
  +-----------------+-----------------+-----------------+-----------------+

                                80386 INTERRUPT GATE
   31                23                15                7                0
  +-----------------+-----------------+---+---+---------+-----+-----------+
  |           OFFSET 31..16           | P |DPL|0 1 1 1 0|0 0 0|(NOT USED) |4
  |-----------------------------------+---+---+---------+-----+-----------|
  |             SELECTOR              |           OFFSET 15..0            |0
  +-----------------+-----------------+-----------------+-----------------+

                                80386 TRAP GATE
   31                23                15                7                0
  +-----------------+-----------------+---+---+---------+-----+-----------+
  |          OFFSET 31..16            | P |DPL|0 1 1 1 1|0 0 0|(NOT USED) |4
  |-----------------------------------+---+---+---------+-----+-----------|
  |             SELECTOR              |           OFFSET 15..0            |0
  +-----------------+-----------------+-----------------+-----------------+
```
关于 Segment Present

> Segment-Present bit: If this bit is zero, the descriptor is not valid for use in address transformation; the processor will signal an exception when a selector for the descriptor is loaded into a segment register. Figure 5-4 shows the format of a descriptor when the present-bit is zero. The operating system is free to use the locations marked AVAILABLE. Operating systems that implement segment-based virtual memory clear the present bit in either of these cases:
> 
> When the linear space spanned by the segment is not mapped by the paging mechanism.
> When the segment is not present in memory.




## TSS （Task Status Segment）
没想到在看似平常的GDT下面，居然出现了一个全新的没见过的概念。。。
https://www.cnblogs.com/yasmi/articles/5198138.html

首先GDT里有TSS Descriptor，用来保存当前指令地址。
另外，在IDT里有Task Gate Discriptor，相当于是TSS的一个指针，不过在权限上可以单独设置
Task Gate Discriptor在LDT里居然也有！
http://www.logix.cz/michal/doc/i386/chp07-04.htm

1. 加载task register
ltr 指令使用提供的 selector 在 GDT / LDT 里索引查找到 TSS descriptor 后，加载到 TR 寄存器里。初始的 TSS descriptor 必须设为 available 状态，否则不能加载到 TR。processor 加载 TSS descriptor 后，将 TSS descriptor 置为 busy 状态。
2. 任务切换
当前进程要切换另一个进程时，可以使用 2 种 selector 进行：使用 TSS selector 以及 Task gate selector（任务门符）。
当前进程的执行环境被保存在当前进程的 TSS segment 中。
发生了 TSS selector 切换。新的 TSS selector 被加载到 TR.selector，而新的 TSS descriptor 也被加载到 TR 寄存的隐藏部分。
processor 从当前的 TSS segment 取出新进程的执行环境。经过相关的 selector & descriptor  的常规检查以及权限检查。通过之后才真正加载。
将新进程的 TSS descriptor 置为 busy 状态。使得新进程不能重入。


> https://www.xuebuyuan.com/737019.html
> 我们可以看到，只有用CALL指令+调用门方式跳转，且目标代码段是非一致代码段时，才会引起CPL的变化，即引起代码执行特权级的跃迁，这是目前得知的改变执行特权级的唯一办法

mark 关于push esp pop esp
https://blog.csdn.net/particleHorizon/article/details/78722683

## 中断门调用过程
最早设置好idt表，通过汇编指令lidt。里面保存了表的大小和起始地址，地址指向的表的每一项都有地址，段寄存器，特权级这三个属性。特权级限制了访问，而地址加上段寄存器的组合就指向了中断服务程序。
于是当发生中断的时候，cpu就会找到指定的地址调用。
调用前，会依次把一些重要的寄存器压栈。
(特权级切换时还有的ss, esp)eflags, cs, eip, error code。
然后就像call调用一样，跳转到指定的地方继续运行。
这里，一般的操作系统会继续保存上下文，以方便恢复一切结束中断。被打断的有当前的各种寄存器，还有栈。结束的时候就恢复自己保存的栈和寄存器再iret就可以了。
当不切换栈的时候是直接保存在当前栈上，反正之后会回来清理，中断的时候cpu忙，旧的栈也没有人来用。
切换栈的时候，系统是会从tss段取出esp吗？保存的栈帧是在系统栈还是在用户栈？？

https://www.cnblogs.com/chaozhu/p/6283495.html
1. 在发生中断、异常时前，程序运行在用户态，ESP指向的是Interrupted Procedure's Stack，即用户栈。
1. 运行下一条指令前，检测到中断（x86不会在指令执行没有指向完期间响应中断）。从TSS中取出esp0字段（esp0代表的是内核栈指针，特权级0）赋给ESP，所以此时ESP指向了Handler's Stack，即内核栈。
1. cpu控制单元将用户堆栈指针（TSS中的ss，sp字段，这代表的是用户栈指针）压入栈，ESP已经指向内核栈，所以入栈指的的是入内核栈。
1. cpu控制单元依次压入EFLAGS、CS、EIP、Error Code（如果有的话）。此时内核栈指针ESP位置见图4中的ESP After Transfer to Handler。

https://www.xuebuyuan.com/915513.html
也是讲堆栈切换的

为什么当前的tss保存的是内核的东西？虽然这和TSS的用法不符，但是应该没错吧。
为什么垃圾intel 80386 程序员手册没有在中断一节说清楚？是我瞎吗

那么当一个os初始化完成之后，它第一个用户态程序是怎么切换权限级的？直接加载一个更低的权限？
当系统解析ELF文件的时候，就要安排好各种数据，分段（有页吗？）设置好权限（读写执行）和权限级。然后跳转过去吗？这样不就直接转换

## 8295A和8253 timer
详解8259A
https://blog.csdn.net/longintchar/article/details/79439466
https://baike.baidu.com/item/8253%E8%8A%AF%E7%89%87/3699917








































































