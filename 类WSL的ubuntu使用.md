---
title: 类WSL的ubuntu使用
date: 2020/1/11 21:46:25
categories:
- CTF
tags:
- CTF
- pwn
---


# 类WSL的ubuntu使用
为了搭建方便可靠的pwn环境，但是为了32位的兼容性，我开始使用vmware + ubuntu server
<!-- more -->

## 为什么要类WSL？
Windows Subsystem for Linux
之前直奔着WSL而去，按照网上说的各种方法，确实可以实现特别不错的windows和linux集成混血体验，让我顺畅运行各种linux的软件
但是缺点是驱动有问题，还是不支持ext文件系统
最严重的缺点就是不能原生运行32的elf了。这挺影响我打pwn的，靠虚拟机运行32位程序，日常使用还可以，想要调试就不太方便了。

但从使用wsl的经历我认识了Xming以及VcXsrv

## 1 VcXsrv
配置好VcXsrv：
右键，属性，兼容性，更改高DPI设置
这里要选高DPI缩放替代，缩放执行：应用程序
勾选上面的修复没有什么用，下面的才有用。
另外启动的时候要选择允许所有连接，这样才能够连上

## 2 vmware的文件共享

设置里面打开文件共享，就可以在/mnt路径里面看到了

自动挂载共享
https://wiki.archlinux.org/index.php/VMware/Install_Arch_Linux_as_a_guest#Shared_Folders_with_vmhgfs-fuse_utility
https://blog.csdn.net/shengerjianku/article/details/89874376

## 3 vscode的远程连接

vscode用ssh连进去，选择挂载的共享文件夹，再在里面做题
仿佛没有虚拟机一样，还是在本地的硬盘上，(*^_^*)

## pwndbg
pwndbg可以连接ida，
https://github.com/pwndbg/pwndbg/blob/dev/FEATURES.md#ida-pro-integration
首先下载对应的ida脚本，再修改对应的远程脚本的host为对应网卡的ip
在ida里面load script file运行。

接下来在虚拟机的pwndbg里面设置
也就是说要改一下config的远程ip地址

在gdbinit里面加上
set ida-rpc-host 192.168.230.1
这样如果ida加载了脚本，就会自动连接上了！！

每次断下来的时候就都会在ida里面标上颜色！！！
而且在ida里面下断点也可以！！