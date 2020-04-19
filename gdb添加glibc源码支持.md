---
title: gdb添加glibc源码支持
date: 2019/9/19 21:46:25
categories:
- CTF
tags:
- CTF
- linux
- libc
---

# gdb添加glibc源码支持

<!-- more -->

学好glibc和<！操作系统！>是我学pwn大法的第一步
推荐阅读：http://blog.chinaunix.net/uid-24774106-id-3642925.html
## 1. 下载符号信息

首先在gdb中使用命令
> info sharedlibrary

查看链接库文件的情况，可以看到有没有加载符号信息

### 经本人试验，ubuntu19.04的libc2.29.so似乎自带调试符号信息！只需要下载源码

可能需要安装的包：
glibc-source
ubuntu有一个库
sudo apt-get install libc6-dbg
描述：GNU C Library: detached debugging symbols
libc6-dev是库，之后通过apt-get source 下载libc代码。
32位还有libc-dbg:i386

> 在Fedora/Red Hat 系的OS上，需要安装的软件包的名字不叫 libc6-dbg，libc6-dev，貌似应该是glibc-debuginfo。
> 来自：https://blog.nlogn.cn/posts/2012-03-27-trace-glibc-by-using-gdb/

gdb相关变量还有
 set debug-file-directory /usr/lib/debug
 来自：https://blog.csdn.net/hejinjing_tom_com/article/details/39155305
 set solib-search-path
 来自：https://blog.csdn.net/xqhrs232/article/details/52854304


## 2. 下载源代码
基本上都是用apt source libc6-dev
麻烦在于要加上apt-src源。
可以放在自己的目录，也有人放在/usr/lib/src目录下，这里经常放linux-header什么的
之后在gdb里使用directory命令引入源码。

## ld.so的调试
我就说gdb是怎么找到调试的符号信息的，没想到根本就没有找，因为自带了。。。
发现ld.so无法调试。
继续研究这个符号信息安装的原理
安装的符号信息在/usr/lib/debug/lib/i386-linux-gnu类似的地方【来自开头的推荐阅读】，是一些有符号表但是不能运行的so文件。
发现似乎可以通过-s选项加载（不过动态库好像不行）
~~symbol-file [ filename ]~~
add-symbol-file filename address
https://blog.csdn.net/dong_007_007/article/details/49247725
https://zhiwei.li/text/2010/01/11/gdb%e6%89%8b%e5%86%8c13-%e6%9f%a5%e7%9c%8b%e7%ac%a6%e5%8f%b7%e8%a1%a8/

成功让ld.so也显示源代码！
果然还是官方手册强！
符号信息安装后还是要自己加载。。。