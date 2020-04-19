---
title: ctf 常用的工具
date: 2019/6/22 20:46:25
categories:
- CTF
tags:
- CTF
- linux
---

# ctf 常用的工具

## 脱壳
<!-- more -->
upx脱壳简单方便
upx -d <文件路径>

## 查看elf文件类型
file <文件路径>
例子：

> wjk@ubuntu:~/ctf$ file bof
bof: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=ed643dfe8d026b7238d3033b0d0bcc499504f273, not stripped

## 查看安全保护
checksec <文件>
> wjk@ubuntu:~/ctf$ checksec ./bof
[*] '/home/wjk/ctf/bof'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled

## linux下的OllyDbg --- EDB
https://github.com/eteran/edb-debugger

## binwalk
 Binwalk被设计用于识别嵌入固件镜像内的文件和代码。

## peda
PEDA - Python Exploit Development Assistance for GDB

## pwndbg
https://github.com/pwndbg/pwndbg
使用GDB 7.7的Ubuntu14.04和使用GDB 7.11的Ubuntu 16.04支持Pwndbg
听说gdb的好处在于可以动态调exp

## winscp
特别好用的向服务器传文件的工具

## xshell

## ROP:
 one_gadget 这个ruby写的工具用到了符号执行，准确率高