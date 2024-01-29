---
title: Openwrt的GRETap6隧道的MTU问题
date: 2023/11/06 11:11:12
categories:
- Dev
tags:
- Networking
---

Openwrt的GRETap6隧道的MTU问题

<!-- more -->

### 问题描述

**背景**：两个发射wifi的路由器用GRETap在链路层组网，我在这边wifi通过SSH连接那边路由器有线连接的台式机，诡异地连不上去。经过调研发现应该是MTU的问题。搜了一下应该是GRETap6的问题。

**诊断**：

- 这个[Reddit帖子](https://www.reddit.com/r/sysadmin/comments/737c1z/friendly_reminder_if_ssh_sometimes_hangs/)提到这种SSH连接停住的现象很可能是MTU问题。确实很可能是MTU的问题。通过直接把笔记本接到那边路由器，或者连那边的wifi，问题都不再出现。
- 这个人也有[IPv6 GRETap的MTU问题](https://www.reddit.com/r/ipv6/comments/pmxg2m/ipv6_mtu_issue_with_hosts_behind_mikrotik_router/)

**Path MTU Discovery (PMTUD)原理**：虽然本地机子的MTU不会变，但是发包后路由器就会返回icmp消息说，你太长了，然后本地机子就会用mtu1300.

但是，关键的问题在于，GRETap不在IP层。。。也就是说，现在在链路层L2组网，我和那边机子中间没有任何其他的中介，直接链路层找它了。虽然中间我们包被中间的路由器由GRE隧道封装了，但是路由器即使知道你笔记本这个包有问题，想和你笔记本说一下，你包太大了，如果通过ICMP的消息说的话，这属于L3层了。在笔记本看来，我是直接和那边机子沟通啊，你怎么突然过来插一脚，根本不会理。

[这个帖子](https://forum.openwrt.org/t/solved-gretap-tunnel-and-mtu-size-problem/14182/8)里和我遇到了相同的问题。想想GRE包在被包裹之后，传递给那边的时候，还是通过IP层传的。[这个帖子](https://forums.gentoo.org/viewtopic-t-1007394-start-0.html
)也尝试详细描述这个问题。

在[内核](https://bugzilla.kernel.org/show_bug.cgi?id=14837 )那边也有相关的讨论：
- 最底下提到了`ignore-df nopmtudisc`这两个选项。[这个回答里](https://stackoverflow.com/questions/42545112/linux-gretap-net-ipv4-ip-gre-c-how-to-set-value-of-key-tun-flags)也有相关的例子
- 中间看到一些，bridge按照标准会丢大于自己MTU的包，这个好像无关，bridge取的是最小的MTU。

### OpenWRT的支持

**Package GRE**：先看看GRE包的[源码](https://github.com/openwrt/openwrt/blob/ae500e62e2938e112ae1fc6aa7389e8c7b784b13/package/network/config/gre/files/gre.sh)吧，稍微看看就可以发现，这里调的是`proto_send_update`，是`netifd-proto.sh`里面的函数。引用到了另外一个仓库：`netifd`。而`netifd`并不是调的ip link命令，而是直接C语言底层实现相关操作。

ip link命令又是怎么实现这个功能的呢？ip这个命令行工具来自`iproute2`包，搜索相关源码可以找到[`link_gre.c`](https://github.com/iproute2/iproute2/blob/0c3400cc8f576b9f9e4099b67ae53596111323cd/ip/link_gre.c#L398)这里。可以发现在`linux/if_tunnel.h`这个内核头文件里有个flag叫`IFLA_GRE_PMTUDISC`。这里操纵的是`struct nlmsghdr`结构体，

那边netifd也是一样，但是使用了[libnl](https://www.infradead.org/~tgr/libnl/doc/core.html)库。[内核文档](https://docs.kernel.org/userspace-api/netlink/intro.html)和[libnl文档](https://www.infradead.org/~tgr/libnl/doc/core.html#core_msg_format)都描述了netlink消息的细节。

然而，[这里](https://github.com/openwrt/netifd/blob/f01345ec13b9b27ffd314d8689fb2d3f9c81a47d/system-linux.c#L4012)可以看到，仅对IPv4的隧道支持了`ignore-df`选项。在[ip命令的帮助](https://man7.org/linux/man-pages/man8/ip-tunnel.8.html)里可以发现，ignore-df选项仅忽略IPv4的Dont Fragment包。

[这里](https://www.reddit.com/r/ipv6/comments/pmxg2m/ipv6_mtu_issue_with_hosts_behind_mikrotik_router/)提到了，在IPv6里根本没有分包，必须由中间节点发icmp6消息提醒发送端减少MTU。很可能这个问题是无解的，只能改自己本地MTU？
