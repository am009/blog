---
title: 安装使用one_gadget
date: 2019/8/17 20:46:25
categories:
- CTF
tags:
- CTF
- tutorial
- ruby
- snap
---

# 安装使用one_gadget

给snap加代理，给gem加代理。
<!-- more -->
最近需要用到one_gadget。
https://xz.aliyun.com/t/2720
去github上搜，除了知名的ruby的one_gadget，还有一个python版本的，在我的ubuntu19.04上使用下来效果不佳

首先通过snap安装ruby，因为apt里的ruby版本太旧了。
snap直接下载速度极慢。特别的是，给snap加代理还不能通过那个环境变量
https://ywnz.com/linuxjc/4761.html
```
 在/etc/environment文件中加入以下两行：

http_proxy=http://[服务器地址]:[端口号]

https_proxy=http://[服务器地址]:[端口号]

然后重启snapd服务，运行以下命令：

sudo systemctl restart snapd

注：snapd读取/etc/environment，因此设置代理环境变量是有效的，在Ubuntu上通过设置→网络→网络代理自动完成的，因此只要在更改该文件后重新启动snapd就可以了。
```

给gem加代理就可以直接
export http_proxy=http://
export https_proxy=http://

安装后不知道在哪里启动
通过find大法
find -name one_gadget
发现是在~/.gem/bin里
于是在~/.bashrc里加入
```
#PATH=$PATH:$HOME/.gem/bin
PATH=$PATH:$HOME/.gem/bin:$HOME/.local/bin

```

