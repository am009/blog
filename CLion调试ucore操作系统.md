---
title: CLion调试ucore操作系统
date: 2019/9/29 13:46:25
categories:
- OS
tags:
- ucore
- jetbrains
---

# CLion调试ucore操作系统
渐渐被jetbrains全家桶侵蚀
<!-- more -->

1. 安装makefile支持的插件
2. 编译（make）
3. make gdb
4. 添加remote gdb debugger，里面文件选择bin/kernel
5. 修改自己的~/.gdbinit,加入source 到自己项目里的gdbinit文件，里面添加两句用于添加符号信息
6. 在编辑器里面点击下断点
7. 启动debug

然后就发现真的停在断点了，哈哈哈哈哈哈哈哈哈哈哈哈哈哈哈
\\(\^o\^)/~
额，真的不用eclipse CDT吗？
没必要了，clion已经可以胜任了!!

另外还需要compiledb，具体请访问
> https://blog.csdn.net/lylwo317/article/details/86673912

首先
compiledb -nf make
再重新打开项目，选中新生成的文件，OK！

## 调试

clion只显示运行过的语句的反汇编？？

注意相关中间文件的生成。
```
@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock): 
使用objcopy将bootblock.o二进制拷贝到bootblock.out，其中：
-S：表示移除符号和重定位信息；
-O：表示指定输出格式；
@$(call totarget,sign) $(call outfile,bootblock) $(bootblock): 
使用sign程序, 利用bootblock.out生成bootblock;
```
摘自别人的lab1实验报告
https://www.jianshu.com/p/2f95d38afa1d
所以，其实bootblock也可以调试的，只要加载到带调试信息的bootblock.o
但是似乎不能同时加载symbol-file bootblock.o和file bin/kernel。不过可以调试的时候通过命令切换
而什么asm，什么sym，其实是objdump打印的反汇编和符号表信息，不是给gdb加载的，而是给自己阅读的。
相关gdb文档：
> http://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html

目前我的gdbinit如下：
```
define hook-stop # 用来自己用gdb调试时方便直接不断步进，观察变化
x/10i $pc
info registers
info stack
end



file ~/ucore_labs/lab1_result/obj/bootblock.o
symbol-file ~/ucore_labs/lab1_result/obj/bootblock.o
file ~/ucore_labs/lab1_result/bin/kernel
# 这个好像可以在clion远程gdb里面选择本地文件的时候就带上了，不用另外添加，可以注释掉试试
directory ~/ucore_labs/lab1_result/

#set arch i8086
#b *0x7c00


#target remote :1234
#break bootmain
#continue

```

我把gdb窗口单独拖出来拉长，方便调试

另外似乎还要注意架构的设置：
> https://chyyuu.gitbooks.io/ucore_os_docs/content/lab0/lab0_2_4_4_6_set_debug_arch.html
> 设定调试目标架构
> 在调试的时候，我们也许需要调试不是i386保护模式的代码，比如8086实模式的代码，我们需要设定当前使用的架构：
> (gdb) set arch i8086
> 这个方法在调试不同架构或者说不同模式的代码时还是有点用处的。








