---
title: libc源码相关阅读
date: 2019/8/17 20:46:25
categories:
- CTF
tags:
- CTF
- rop
---

# libc源码相关阅读

<!-- more -->


https://www.cnblogs.com/lidabo/p/5344777.html

我原本以为研究这种什么libc怎么写的这些玩意的人很少，没想到有书会写这种东西。
APUE：Unix环境高级编程
赶紧收藏


## system

相关链接
https://blog.csdn.net/u010039418/article/details/77017689
https://www.cnblogs.com/lidabo/p/5344777.html

首先下载下来glibc的源码
system调用do_system，位于/sysdeps/posix/system.c
放在sysdeps文件夹里面也就是说system函数其实是依赖于平台的吗
除了一些信号相关的东西，其实主要还是fork再execve


## libc_start_main相关
主要关注的问题是，为什么重新返回main函数，溢出的偏移就变了？这个问题之前碰到了，下次碰到再来看看吧。