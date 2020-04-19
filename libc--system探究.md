---
title: libc--system探究
date: 2020/1/15 20:46:25
categories:
- CTF
tags:
- libc
- CTF
---


# libc--system探究
<!-- more -->

自然是因为有的时候调用system出问题，从而继续探究
为什么有的时候rop调用system会报错？

https://www.xmcve.com/2019/05/%E5%9C%A8%E4%B8%80%E4%BA%9B64%E4%BD%8D%E7%9A%84glibc%E7%9A%84payload%E8%B0%83%E7%94%A8system%E5%87%BD%E6%95%B0%E5%A4%B1%E8%B4%A5%E9%97%AE%E9%A2%98/

https://blog.csdn.net/hu19921016/article/details/80456753

----


https://stackoverflow.com/questions/54393105/libcs-system-when-the-stack-pointer-is-not-16-padded-causes-segmentation-faul

里面提到The x86-64 System V ABI guarantees 16-byte stack alignment before a call, so libc system is allowed to take advantage of that for 16-byte aligned loads/stores.
看来又错过了一本重要的标准

找这个标准
https://stackoverflow.com/questions/18133812/where-is-the-x86-64-system-v-abi-documented
找到了
https://github.com/hjl-tools/x86-psABI/tree/hjl/master

下载下来看第18页，里面的图显示了需要在调用函数时对齐16字节。
也就是call的时候的push rip占了8字节，然后函数开头保存ebp占用8字节，刚好16字节。
所以只要遵循了这个函数调用就可以正常使用system函数了?

另外这个system V规范也是定义ELF文件的规范，看来有必要好好看下了

## 再次尝试
首先保证了栈是对齐的，这次没有报什么汇编指令执行失败的错误了
但可惜，还是没能get shell，好像是调用的程序退出了吗？

仔细观察发现是溢出太多，导致环境变量被溢出了，导致调用execve的时候存了一个非法地址到原本环境变量rdx指向的地方。。。
如果那里存的是0还好，如果是什么非法的鬼地址就惨了，调用失败。


