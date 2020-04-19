---
title: 树莓派resbian
date: 2019/7/31 20:46:25
categories:
- 树莓派
tags:
- raspberrypi
- raspbian
---

# 树莓派resbian
<!-- more -->
2019-12-07
https://github.com/openfans-community-offical/Debian-Pi-Aarch64/blob/master/README_zh.md
这是新系统？好像还不错
## 上手系统安装
下载系统镜像，解压。。。
一般发行版下载的系统的iso，但是树莓派系统是img的压缩包，要先解压。

刷镜像一般用win32diskimager。linux用dd命令。
也就是说，如果你经常刷系统，拿到树莓派如果先不急着拓展文件系统空间，安装完一些软件，就可以拿回来用win32diskimager读img镜像出来留着。
刷完反射性地在boot的根目录下新建ssh空文件。
然后就是启动，一句sudo raspi-config打开vnc。
树莓派还是对桌面玩家友好😭，图形界面比命令行调乱七八糟的参数好得多。

一些闲话：
回忆起很久以前ubuntu等桌面版还不知道用什么刷到优盘的时候，有个opensusu的小工具也是刷优盘镜像的。主要需要对磁盘的二进制读写能力，才能写出启动盘。
最近装了一次win10，以前装win7还会用用ultraISO的写入硬盘镜像到优盘，现在它已经落后变得垃圾了，写入不是卡死就是不能启动，现在用rufus-3.6p.exe这个小巧的程序特别强。装系统的时候才了解到难道uefi已经支持ntfs了吗
已经是第二次不能从sd卡启动了。。。是不是uefi歧视sd卡。。。只能用优盘，也不知道是不是驱动不一样。


## journalctl
直接执行，进行日志的查看
参数：
-n 3
查看最近三条记录
-perr
查看错误日志
-overbose
查看日志的详细参数
--since
查看从什么时候开始的日志 
--until
查看什么时候截止的日志

这条命令主要是因为我之前用手机充电器，这里会报低电压警告
最近买了新电源，用这个命令来看看还会不会警告。果然换了电源就看不到了

## 电源管理
sudo iw dev wlan0 set power_save on|off
这条命令关闭wifi的电源管理，否则wifi不稳，我以前通过wifi进行ssh不可靠就是因为这个和电源

## 安装无线网卡驱动
手头有一块垃圾的tenda U6，是rtl8192eu的，还有一张rtl8812au
https://github.com/Mange/rtl8192eu-linux-driver
https://github.com/aircrack-ng/rtl8812au
按照github的readme进行设置，
我有一次只是安装rtl8192eu的dkms模块，没有进行教程接下来的设置操作，就开不了机，只能重刷系统😭。
开机就可以执行
```
sudo apt-get install git raspberrypi-kernel-headers build-essential dkms;
```
让它慢慢下载

## networkManager
这里我需要给树莓派固定的ip，但是发现这次树莓派每次启动wifi网卡mac地址都随机化，每次一个新ip，还查不出这个网卡是哪个厂商的。。。
但是图形化界面没有设置的地方，还是ubuntu的network-manager-gnome友好，里面有这个选项。
于是试图使用network-manager-gnome管理无线网
本来以为network-manager-gnome是gnome桌面专用的东西，后来装了才发现lxde照用不误，有桌面环境就行。
sudo apt-get install network-manager-gnome
这里安装后菜单就出现了advanced network Configuration
修改配置文件
sudo nano /etc/NetworkManager/NetworkManager.conf
```
[main]
plugins=ifupdown,keyfile

[ifupdown]
managed=false
```
把false改成true
#注：managed这里表示是否管理/etc/network/interfaces里配置了的网络接口
修改/etc/network/interfaces，在末尾加上如下内容
```
allow-hotplug wlan0
iface wlan0 inet dhcp
        hwaddress b8:27:eb:00:00:00
```
相关原理：
下面基于2019年7月31日下载的raspbian系统
树莓派的这个发行版的网络组件和其他发行版不一样。一般linux首先读取/etc/network/interfaces，剩下没有被管理的网络接口被network-manager管理。
树莓派的网络也使用了networkmanager，而不使用interfaces文件，下面是树莓派默认的/etc/network/interfaces
```
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

```
而它include的/etc/network/interfaces.d是一个空文件夹，所以raspbian是不用interfaces的。
任何修改/etc/network/interfaces的教程现在都已经过时了，修改该文件会造成兼容性问题，具体表现在桌面右上角的图标失灵，不能正常管理网络。
桌面右上角的网络管理的图标比较简陋，软件包名称是lxplug-network，根据名字可以知道，这个应该是lxde的插件，不过肯定是用在raspbian的pixel桌面上的，而且lxplug- 还是一个系列，还包括蓝牙什么的，它在github上的链接是
https://github.com/raspberrypi-ui/lxplug-network
听说原开发者已经退休了。。。
当右上角的图标失灵时，显示Connection to dhcpcd lost. 点击显示No wireless interface found.
这里可以看出它管理网络使用的是dhcpcd，c代表client，d是deamon守护进程。
下面这个帖子在小白互助的时候，一位大神出来说了一句技术内幕。看来有关树莓派的问题还是到官方论坛搜索比较好，接触全球的帖子。
https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=242721&p=1482723&hilit=%2Fetc%2Fnetwork%2Finterfaces+mac+random#p1482723
其中关键的一句：
dhcpcd has a wpa_supplicant hook in Stretch.
Stretch是Debian9的代号现在都到debian10 buster了。
也就是说，这个管理程序还是用的是wpasupplicant，不是让你去interfaces里点名用wpasupplicant，而是它在dhcpcd里面它自己调用！用wpagui也可以管理到！
至于静态ip，树莓派的桌面环境可以设置。而且如果想在配置文件配置也是在dhcpcd.conf里配置。
https://www.jianshu.com/p/bd918ef98a4d


其他相关页面
https://wiki.lxde.org/en/LXNM
https://wiki.lxde.org/en/LXDE-Qt





## 换tuna源
虽然树莓派基金会的镜像也能用，但是还是清华的快一些。
https://mirrors.tuna.tsinghua.edu.cn/help/raspbian/


## dump镜像
树莓派还是不要用太大的sd卡，完全没必要。现在因为不想以后换系统的时候重新配置文件，现在打算直接把整个sd卡做个镜像压缩一下。8g其实完全足够了。我用的32g的卡，为了省空间，打算先压缩分区大小再dump出来。

1. fsck磁盘检查
要把树莓派关机，把卡拿下来，去别的linux系统上检查
sudo fsck /dev/sda2 -f
使用-f强制检查但是我检查了，还是后面报有node不对，忘了之前是怎么搞的了
2. 使用diskgenius压缩分区。
gparted还真的不容易做压缩分区，看来还是diskgenius好啊

一不小心用了这个感觉挺危险的办法：
https://access.redhat.com/articles/1196333
https://askubuntu.com/questions/780284/shrinking-ext4-partition-on-command-line
还要删除分区再建立。。。

之后构建img文件还是使用win32diskimager，没办法，没什么其他好软件。
不过可以勾选只备份已有分区，挺好，没想到小工具能做到这么实用，我要是也能写出这样的工具就好了。


