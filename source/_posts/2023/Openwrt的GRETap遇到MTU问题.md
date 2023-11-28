---
title: Openwrt的GRETap遇到MTU问题
date: 2023/11/06 11:11:12
categories:
- Dev
tags:
- Networking
---

一个给Openwrt贡献patch的机会

<!-- more -->

### GRETap的问题

**背景**：两个发射wifi的路由器用GRETap在链路层组网，我在这边wifi通过SSH连接那边路由器有线连接的台式机，诡异地连不上去。经过调研发现应该是GRETap的问题。

**诊断**：正如这个Reddit帖子所说：[Friendly reminder: If ssh sometimes hangs unexplainably, check the mtu to the system](https://www.reddit.com/r/sysadmin/comments/737c1z/friendly_reminder_if_ssh_sometimes_hangs/)。确实很可能是MTU的问题。通过直接把笔记本接到那边路由器，或者连那边的wifi，问题都不再出现。

**初步的尝试**

1. 防火墙，lan zone，勾选MSS clamping
2. Interface wlan1，设置MTU为探测的大小（1300）

**Path MTU Discovery (PMTUD)原理**：虽然本地机子的MTU不会变，但是发包后路由器就会返回icmp消息说，你太长了，然后本地机子就会用mtu1300.

但是，关键的问题在于，GRETap不在IP层。。。也就是说，现在在链路层L2组网，我和那边机子中间没有任何其他的中介，直接链路层找它了。虽然中间我们包被中间的路由器由GRE隧道封装了，但是路由器即使知道你笔记本这个包有问题，想和你笔记本说一下，你包太大了，如果通过ICMP的消息说的话，这属于L3层了。在笔记本看来，我是直接和那边机子沟通啊，你怎么突然过来插一脚，根本不会理。

[这个帖子](https://forum.openwrt.org/t/solved-gretap-tunnel-and-mtu-size-problem/14182/8)里和我遇到了相同的问题。想想GRE包在被包裹之后，传递给那边的时候，还是通过IP层传的。[这个帖子](https://forums.gentoo.org/viewtopic-t-1007394-start-0.html
)也尝试详细描述这个问题。

在[内核](https://bugzilla.kernel.org/show_bug.cgi?id=14837 )那边也有相关的讨论：
- 最底下提到了`ignore-df nopmtudisc`这两个选项。[这个回答里](https://stackoverflow.com/questions/42545112/linux-gretap-net-ipv4-ip-gre-c-how-to-set-value-of-key-tun-flags)也有相关的例子
- 中间看到一些，bridge按照标准会丢大于自己MTU的包，这个好像无关，bridge取的是最小的MTU。

### 如何在OpenWRT里实现

**Package GRE**：先看看GRE包的[源码](https://github.com/openwrt/openwrt/blob/ae500e62e2938e112ae1fc6aa7389e8c7b784b13/package/network/config/gre/files/gre.sh)吧，稍微看看就可以发现，这里调的是`proto_send_update`，是`netifd-proto.sh`里面的函数。引用到了另外一个仓库：`netifd`。而`netifd`并不是调的ip link命令，而是直接C语言底层实现相关操作。。

总之，抓住这一点：netifd不支持`ignore-df nopmtudisc`这两个选项，就可能可以给他们交patch吧？应该。
