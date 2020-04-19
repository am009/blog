---
title: 需要巩固的ctf知识
date: 2019/8/8 20:46:25
categories:
- CTF
tags:
- CTF
- heap
- 线下
---


# 需要巩固的ctf知识

## Partial RELRO
Partial RELRO is the default setting in GCC, and nearly all binaries you will see have at least partial RELRO.

From an attackers point-of-view, partial RELRO makes almost no difference, other than it forces the GOT to come before the BSS in memory, eliminating the risk of a buffer overflows on a global variable overwriting GOT entries.



## 动态链接
那张gif实在是太强了，详细又完善，但是我一直写博客懒得贴图。以后再上图
在调用动态链接的函数的地方，实际上还是call的那个函数，这样和其他本程序内函数在调用形式上统一。
其实call只是一种能通过retn跳回来的jmp，功能上和jmp类似的，但是call调用函数的功能让人感觉过于完善，而jmp则更简单直接。
这里发现，.plt段是指令段，位于text段的低地址处，而。.got.pllt段则是数据段，比data段的数据低。
动态链接的函数会先通过call调转到plt段。这就是在调用自己的函数和调用动态链接的函数的中间层，它向上提供了和调用自己函数一样的接口（直接call），而且堆栈寄存器怎么样了。它在得到控制权的时候，plt表里上来就是一个jmp，依据.plt.got里面保存的地址进行跳转，跳转到真实的地址。opcode有6字节。如果是延迟绑定，这里保存的就是下一条指令的地址，也就是说，此时的jmp就类似nop一样，继续向下执行。下面是push 序号（5字节），jmp到plt表开头的函数，然后进行解析。也是5字节。
为什么我要列举上面的字节数（在64位条件下）因为这里加起来正好是令人惊讶的16字节，直接就对齐了！！！
plt表开头的函数便push了一下linkmap就调用dl_resolve了。即调用_dl_runtime_resolve(link_map_obj, reloc_index)
这里的link_map似乎就是got表的起始地址
进一步学习可以去学习ret2dl_resolve了。
got表是干什么用的？那.got.plt那

动态链接的解析听说之依赖于给定的字符串！
动态链接和多线程有没有关系？

## 使用ROPgadget
这个内置在pwndbg中，真是太强了，ropper也内置了！！！
https://github.com/sashs/Ropper
https://github.com/JonathanSalwan/ROPgadget
ropper的commit更多，但是只有975 star，但是ropgadget的star有2.2k


## 地址映射
64位不开pie的情况下
在0x40 00 00处加载程序
在0x60 00 00处加载数据，这里前面有0x1000的数据在运行时是只读的。
从0x60 10 00处开始是got表，和程序的data段，bss段。
大概小一点的程序会映射到0x60 20 00。也就是有0x1000 4096字节。。。而程序可能只用了几百字节，剩下的两三千字节可以用作栈迁移。ROP如果大量使用pop的gadget会使栈不断下降。复杂的函数调用，栈也会增长很快。

## 堆
堆块大小计算：
libc只会重定位中间几位，所以，vmmap的地址和libc重定位的地址存在固定偏移。
<!-- more -->
不同大小的堆在什么样的bin里？
\__free_hook在.bss段，从.got跳转过去
\__malloc_hook和__realloc_hook在.data段（有初始值）


对rsp+? 有要求的one_gadget可能要多尝试几次才会成功

## dbg相关
1. 下断点要用基址（用vmmap看）+ida里的地址才是程序真正的虚拟地址


## exp相关
context里可以加上timeout，这样不容易卡住

## 线下赛相关
批量打人的时候注意拿完flag关闭和别人题目的端口的p=remote(···)连接
p.close()

## Asis_2016_b00ks
（Asis_2016_b00ks）在做的时候，如果one_gadget没有满足条件，它会疯狂打印菜单。。。
这道题，关键的漏洞点在于bss段author_name在book前面。导致author_name会覆盖一个null到第一本书的末尾字节。这里如果先写author_name,再写指针，可以导致末尾null被覆盖，打印author_name可以泄露指针。如果再写author_name，可以让指针偏离位置，最好是偏到description上。
然而能否偏到description上也不知道。description完全不知道具体地址，只有泄露的地址。只能根据malloc的先后顺序估测。也就是description在malloc book结构体之前，大概在book指针-0x30的位置处。

## 汇编基础

repe cmpsb https://blog.csdn.net/fulinus/article/details/8277442
__IO_getc就是getchar https://baike.baidu.com/item/getc
strtoul https://baike.baidu.com/item/strtoul/1671982?fr=aladdin
fread与fget https://blog.csdn.net/summerlemon/article/details/1341118

## libc泄露相关
https://www.jianshu.com/p/8d2552b8e1a2
在线网站：
https://libc.blukat.me/
libcSearcher
https://github.com/lieanu/LibcSearcher
就是调用libc-database的，而且特别简单，只要把libc-database放到旁边就可以了
干脆ln -s软链接两个文件过来吧
libc-database的安装
直接zip安装然后get就好吧，没想到libc-database太强了，最新的ubuntu版本号20.04的focal都放上去了！完全不需要自己手动去修改。而且还支持了debian的！！！
这里面14.04 trusty的包已经下载不到了，连ubuntu镜像都不再支持了。。。
https://github.com/niklasb/libc-database/pull/13
packages.ubuntu.com remove trusty support on 2019.04.23
这里下载使用的是wget，设置代理在/etc/wgetrc里面设置。。。
export的方法设置代理感觉总是不管用，不如在配置里面强制使用代理
https://www.thegeekdiary.com/how-to-use-wget-to-download-file-via-proxy/
