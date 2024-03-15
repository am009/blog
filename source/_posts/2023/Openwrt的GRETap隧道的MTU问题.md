---
title: Openwrt的GRETap6隧道的MTU问题
date: 2024/03/10 11:11:12
categories:
- Dev
tags:
- Networking
---

Openwrt的GRETap6隧道的MTU问题

<!-- more -->

## 背景

两个发射wifi的路由器用GRETap通过IPv6在链路层组网，我在这边wifi通过SSH连接那边路由器有线连接的台式机，诡异地连不上去。经过调研发现应该是MTU的问题。

### IPv4和IPv6的MTU处理

MTU是指最大传输单元。当数据包在互联网上通过路由经过各种各样的链路的时候，不同链路能传的最大包大小不同，包太大就需要被分片或者丢弃。分包都会造成性能开销，所以最合适的包大小当然是整个链路里面最小的那个包大小。

IPv4同时允许分片和Path MTU Discovery(PMTUD)。

- PMTUD就是指路由器直接把包丢了，向发送端发个icmp消息说包太大了，你重新按这个大小发包吧。
- 分片就是指中间的路由器可以通过设置Fragment标记进行分片，将一个包直接拆成两个包。

IPv6在设计时直接完全禁止了分片的存在，没有分片标志位了。包如果太大只能被丢弃，然后使用PMTUD通知发送端。

## 局域网直接访问的问题

GRE隧道技术，通过在IP层里封装链路层的包，实现了二层组网。经过封装的包会出现这样的层次结构。

以太网 --(封装)--> IP --(封装)--> GRE --(封装)--> 内层以太网 --(封装)--> 内层IP --(封装)--> ...

同时GRE隧道不支持分片，因此原本能传的大包，经过隧道之后因为多封装了外侧的以太网+IP+GRE头，包会变大，导致超出MTU。

而且如果在隧道中间丢包，则PMTUD也无法正常工作。关键的问题在于，GRETap不在IP层。也就是说，现在在链路层L2组网。如果我要ssh连接隧道另一头的机子，连接时直接通过链路层，中间没有任何的路由。虽然实际上我们包被中间的路由器用GRE隧道封装了，但是路由器即使知道你笔记本这个包太大了传不了，想和你笔记本说一下，说你包太大了，如果通过ICMP的packet too big消息说的话，这属于L3层了。在笔记本看来，我是直接和那边机子沟通啊，你怎么突然过来插一脚，根本不会理会。

推荐阅读：[RFC 7588 GRE隧道的分片方案](https://www.rfc-editor.org/rfc/rfc7588.html)。这篇RFC介绍了各大企业在使用GRE隧道时遇到分片问题时，由于默认的协议不支持分片，从而设计自己的方案，并对协议做出拓展的事情。

## IPv4时的GRETap解决方案

Linux内核的GRE隧道，最近支持了`ignore-df nopmtudisc`选项。（如果是IPv4封装的GRE）在最外层的IP包处强制允许分片。这样即使单个包太长，中间的路由器也可以直接分片传输。

然而，因为IPv6完全不支持包分片，IPv6的GRE隧道就不能这么做了。

## 跨交换机GRETap6组网

上面说的都是使用GRE通过互联网创建隧道。但是实际上我们想做的是通过IPv6，跨学校交换机在两个房间组网。如果经过交换机的这个链路支持很大的MTU，那我们隧道就应该没问题。经过测试，交换机能传输大的以太网包。但是我们发现，怎么改接口的MTU都不管用。

不断抓包，使用nping命令直接发送以太网包发现，IPv6和IPv4的MTU居然不一样。先把网卡MTU配置为2000。首先使用`nping -c 1 --icmp --dest-mac 00:0C:29:72:2D:59 1.1.1.1 --data-length 1954` 可以发出总大小为2000的以太网包。但是如果转而想发送IPv6，使用 `ping -s 11472 -Mdo 2001:250:4000:511d:114:514:1919:810` 提示`ping: local error: message too long, mtu: 1500`。

进一步搜索找出了问题的根源：IPv6的RA路由通告会通告当前网段的MTU大小。通过设置静态地址，忽略RA，可以使得MTU正确地变成网卡配置的MTU。

### 其他

**诊断**：

- 这个[Reddit帖子](https://www.reddit.com/r/sysadmin/comments/737c1z/friendly_reminder_if_ssh_sometimes_hangs/)提到这种SSH连接停住的现象很可能是MTU问题。确实很可能是MTU的问题。通过直接把笔记本接到那边路由器，或者连那边的wifi，问题都不再出现。
- 这个人也有[IPv6 GRETap的MTU问题](https://forum.vyos.io/t/ip6gre-and-fragmentation/11710)

**Path MTU Discovery (PMTUD)原理**：虽然本地机子的MTU不会变，但是发包后路由器就会返回icmp消息说，你太长了，然后本地机子就会用mtu1300.

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

[这里](https://www.reddit.com/r/ipv6/comments/pmxg2m/ipv6_mtu_issue_with_hosts_behind_mikrotik_router/)和[这里](https://forum.vyos.io/t/ip6gre-and-fragmentation/11710)提到了，在IPv6里根本没有分包，必须由中间节点发icmp6消息提醒发送端减少MTU。很可能这个问题是无解的，只能改自己本地MTU？

这个[RFC文档](https://www.rfc-editor.org/rfc/rfc7588.html)提到了一些实现，有些厂商为了解决这个问题而手动实现了分包。
