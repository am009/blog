---
title: Linux kernel env
date: 2019/9/25 11:46:25
categories:
- CTF
tags:
- CTF
- kernel
- linux
---


# Linux kernel env

https://beyermatthias.de/blog/2016/11/01/setup-for-linux-kernel-dev-using-qemu/
https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/
<!-- more -->

下载linux主线内核代码：
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
没想到这么大，1.36g了，还好有tuna，飞一般的下载速度
这个下载的就是主线版本，因为直接打开kernel.org，点击mainline版本右边的browse，打开的就是如下的链接：
> https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/?h=v5.3
和那个教程的网站克隆的一样。
没想到编译挺快的
记得根据第二个链接加上调试信息和gdb调试脚本

## busybox
> curl https://busybox.net/downloads/busybox-1.30.1.tar.bz2 | tar xjf -


## 启动
qemu-system-x86_64这个软件包不知道为什么没了(ㄒoㄒ)
找了半天，没想到是ubuntu自带了
运行时报错
```
wjk@wjk:/tmp/initramfs/x86-busybox$ qemu-system-x86_64 \
>     -kernel arch/x86_64/boot/bzImage \
>     -initrd /tmp/initramfs-busybox-x86.cpio.gz \
>     -nographic -append "console=ttyS0"
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
qemu: could not load kernel 'arch/x86_64/boot/bzImage': No such file or directory
```
好吧，只是找不到目录，不过修改目录到刚才编译的Linux文件夹，启动成功了！

## helloworld
为了简单轻松入门，我选择先学习调试，调试helloworld一波
还是使用第一个教程中的，只不过不去编译filesystem，而是编译一个helloworld玩玩
在内核目录下新建自己的文件夹，新建Kconfig文件，内容。。。
```
config HELLO_WORLD
        tristate "HELLOWORLD support"
        help
          Private HW for playing and learning kernel programming.
```
新建helloworld.c 
```
#include <linux/init.h>
#include <linux/module.h>

static int __init helloworld_init(void)
{
        printk("cool module loaded\n");
        return 0;
}

static void __exit helloworld_fini(void)
{
        printk("cool module unloaded\n");
}

module_init(helloworld_init);
module_exit(helloworld_fini);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("<YOUR NAME GOES HERE>");
```
新建Makefile
```
#
# Makefile for the linux helloworld module.
#

obj-m += helloworld.o
```
obj-m表示编译成模块，obj-y表示编译到内核内
似乎可以写成obj-$(CONFIG_HELLO_WORLD)之类的，然后使用menuconfig之类的设置

反正，现在只要make M=helloworld就行了，得到ko文件

推荐阅读：
> http://emb.hqyj.com/Column/7565.html


## 创建硬盘，方便交换文件

打算把这个磁盘文件保存下来
```
dd if=/dev/zero of=./disk bs=4M count=100
sudo mkfs.ext4 ./disk
mkdir ./disk-mount
sudo mount ./disk ./disk-mount
cp ./linux/wjk/helloworld/helloworld.ko ./disk-mount/
sudo umount ./disk-mount
```

启动qemu带上 -hda ./disk选项, 启动后的系统就有/dev/sda了

```
qemu-system-x86_64 \\
     -kernel ./linux/arch/x86_64/boot/bzImage \\
     -initrd ./initramfs-busybox-x86.cpio.gz \\
     -nographic -append "console=ttyS0" \\
     -hda ./disk
```
```
# In the booted VM
mkdir sda
mount /dev/sda ./sda
ls sda
```
```
/ # insmod ./sda/helloworldk.ko 
[   47.813737] helloworld: loading out-of-tree module taints kernel.
[   47.820481] cool module loaded
/ # 
```
## 其他方法
另外还有其他的安装方式：
https://bbs.pediy.com/thread-226139.htm

## 学习资源
没想到资源有这么多，只是我不知道。。。
https://www.baidu.com/s?ie=UTF-8&wd=kernel%20exploit
内核调试要点：
http://advdbg.org/blogs/advdbg_system/search.aspx?q=%E5%86%85%E6%A0%B8%E8%B0%83%E8%AF%95&p=1
看雪上有一系列文章
https://bbs.pediy.com/search-linux_20kernel.htm
这个网站kernelnubies有点神奇
https://kernelnewbies.org/Documents
