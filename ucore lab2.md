---
title: ucore lab2
date: 2019/9/29 13:46:25
categories:
- OS
tags:
- ucore
---

# ucore lab2
继续看Intel 80386 Programmer's Reference Manual, 1987 (HTML)
http://www.logix.cz/michal/doc/i386/
再看Intel® 64 and IA-32 Architectures Software Developer Manuals
https://software.intel.com/en-us/articles/intel-sdm
<!-- more -->

### 内存管理

分段机制启动、分页机制未启动：逻辑地址--->段机制处理--->线性地址=物理地址
分段机制和分页机制都启动：逻辑地址--->段机制处理--->线性地址--->页机制处理--->物理地址

2^32 = 2^10 * 2^10 * 4k
段机制的时候，limit域长20bit，如果以4k为单位，那么就是最大4g
页机制，一个页4k字节，4字节一个页，一个页可以放1k个页表，那就是1k × 4k = 4M的内存，页表目录则是保存1k个页表，正好4M × 1000 = 4G

内存分配有几个阶段
https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_3_5_4_maping_relations.html

### lab2 宏观内存分配
![](https://segmentfault.com/img/remote/1460000009450843)
图片来自https://segmentfault.com/a/1190000009450840
```
  memory: 0009fc00, [00000000, 0009fbff], type = 1.
  memory: 00000400, [0009fc00, 0009ffff], type = 2.
  memory: 00010000, [000f0000, 000fffff], type = 2.
  memory: 07ee0000, [00100000, 07fdffff], type = 1.
  memory: 00020000, [07fe0000, 07ffffff], type = 2.
  memory: 00040000, [fffc0000, ffffffff], type = 2.
```
可用内存有两块，一个是从00000000 --> 0009fc00 约1M字节
5个十六进制位是20bit，2^10 是1KB,那么2^20就是1M

bios加载bootloader
bootblock占512字节 = 0x200.占0x7c00 ---> 0x 7dff
bootloader探测的内存信息位于0x8000

内核加载到100000，恰好是第二块可用内存的起始地址
第二块内存有07ee0000，大概126MB多一点
在memlayout.h
```
#define KMEMSIZE            0x38000000                  // the maximum amount of physical memory
```
这里限制的物理内存接近1g

然后就是kernel的ELF一路加载下来
其中data段有页目录表（约0x0010b000）和第一个页表
映射0~4MB的首个页表已经填充好。

0x10e000之后就是Page结构体，有32736个，每一个管理4K内存，和内存总量相符。
每一个Page有20字节，所以Pages占了654720字节 = 0x9fd80。占了640K...比想象中大好多
page管理物理内存，数量固定，页表数量可能变化，按需创建。
0x1add80之后就是用来分配的内存了，我们的kernel总共用了0xadd80=712KB。

如果内存再大的话，如果想对应上整个4g空间，那么Page占的空间还可以大32倍。

### diff lab2 with lab1
https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_2_2_phymemlab_files.html
简单来说变化不大，变化的部分除了一些不重要的库（cprintf，什么console的命令解析）其他的基本上都要在写lab2的时候碰到，指导书也讲得很详细。
可以先git commit lab1_result 之后再把lab2 覆盖进去看看变化。

### 内存探测
bootasm.S
添加了有关内存分布的汇编代码
https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_5_probe_phymem_methods.html

### do while(0)
kern/sync/sync.h
https://www.jianshu.com/p/99efda8dfec9

### kern_entry
https://segmentfault.com/a/1190000009450840
https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_3_5_4_maping_relations.html

### gdb打印变量不对
https://blog.csdn.net/jeff_/article/details/53333154
http://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html
CFLAGS	:= -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
修改成
CFLAGS	:= -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gdwarf-4 -nostdinc $(DEFS)
目测没有什么其他问题
然而
造成了print_stackframe()不能打印出行数了


### 页表映射到自己
页目录表地址意义： 这个地址开头的线性地址，管理它的页表在哪一个内存页（物理地址）？
页表地址意义： 你这个4M的地址范围空间，里面的这个地址页在哪个物理内存页？

如果把页目录表项当成页表项，也就是说，里面的每一个页，都是一个页表。
把0xFAC（10 bit）映射到自己，也就是说在0xFAC（10 bit），这4M的虚拟地址，之后加上10bit的页表的虚拟地址前10bit作为index，就可以直接访问到页表了。
这就相当于必须要解两次地址，消耗了一次，这样就可以达到只取一次地址的效果了。

### invlpg 
https://blog.csdn.net/cinmyheart/article/details/39994769
TLB 页表缓冲，我还以为只是页目录表缓冲。。。

问： TLB里面有页目录表吗？
暂时答：因为缓冲一次就缓冲了整个页表，是否命中的判断就是根据对应的页目录表项对不对，所以可能命中缓冲时完全不需要访问页目录表。。。

> 5.2.5 Page Translation Cache
> For greatest efficiency in address translation, the processor stores the most recently used page-table data in an on-chip cache. Only if the necessary paging information is not in the cache must both levels of page tables be referenced.
> 
> The existence of the page-translation cache is invisible to applications programmers but not to systems programmers; operating-system programmers must flush the cache whenever the page tables are changed. The page-translation cache can be flushed by either of two methods:
> 
>   1.  By reloading CR3 with a MOV instruction; for example:
> 
>       MOV CR3, EAX
> 
>   2.  By performing a task switch to a TSS that has a different CR3 image
>       than the current TSS. (Refer to Chapter 7 for more information on
>       task switching.)

When to do or not do INVLPG, MOV to CR3 to minimize TLB flushing
https://www.e-learn.cn/content/wangluowenzhang/626493
TLB的那些事儿
https://blog.csdn.net/omnispace/article/details/61415935

## 缺页？
问题： 现在管理的只是物理内存，那谁管理页表的映设关系？如果某虚拟地址没有对应的页怎么办？
还是说lab2其实就没有管这一块？
暂时答：似乎确实不管这回事？虽然有一些维护页表项的函数，但是没怎么被调用。
先写写lab3吧。



### mark-收藏夹
https://www.jianshu.com/p/abbe81dfe016
https://blog.csdn.net/hezxyz/article/details/95764158





































