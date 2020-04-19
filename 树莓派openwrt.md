---
title: 树莓派openwrt
date: 2019/7/31 20:46:25
categories:
- 树莓派
tags:
- raspberrypi
- raspbian
---


# 树莓派openwrt
<!-- more -->

总是担心树莓派性能不够的问题。挂硬盘还是挂bt还是挂privoxy。感觉以前总是为了易用性舍弃性能，现在是时候好好学学了。嵌入式系统就该有嵌入式系统的样子，而不是总是待机着一个图形界面。

下载factory镜像。sysupgrade镜像是升级用的？
sysupgrade.bin+空闲空间+系统的配置空间=factory.bin的大小

真是太爽了各种软件随便下，体积超小，把多出来的空间全部用来smb哈哈哈哈哈哈

## 热点
为了能够图形界面搭建热点，而不是折腾hostapd。。。

这篇文章和我的困难一样
https://www.jianshu.com/p/d3b690ed56df
所以放弃resbian，转向ubuntu mate或者openwrt
于是开始玩openwrt
听说lede和opwnwrt合并了。听说lede就是一群觉得openwrt开发更新太慢的人搞出来的。
刷入openwrt的镜像选择factory的，sysupgrade使用的包解压时会提示压缩包后有额外内容，为了路由器用于升级的固件的额外验证信息（话说7-zip你就不会无视吗，就是不让我解压）（openwrt真是小巧啊。）

现在也不需要网络共享了，首先不连WiFi登陆，改eth0的ip地址，因为家里的ip就是192.168.1.1，避免冲突
然后安装网卡驱动，支持的网卡只要安装对应的软件包就可以。
然而它不支持我的网卡，宣布安装网卡驱动失败。。。openwrt的网关设置界面叫做luci。新的驱动需要特定代码为这个网关配置页面提供接口，所以单有linux的电脑的网卡驱动好像不够。。。
但是如果安装好驱动，表现起来也是一个wlan的interface，说不定可以配置。。。
不知道是怎么样的
