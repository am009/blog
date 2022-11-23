---
title: 锐捷校园网IPv6的／64内网中继配置：ndp-proxy、relay
date: 2022/11/23 11:11:12
categories:
- Hack
tags:
- Openwrt
- Networking
- Embedded
---

锐捷校园网IPv6的／64内网中继配置：ndp-proxy、relay

<!-- more -->

### 问题描述

#### 锐捷校园网认证

在校园网网页认证的情况下，IPv4应该是只能有一个的，但是IPv6可以有多个。因为现代设备会利用ipv6巨大的地址段，自动在/64子网下生成一个临时ip地址。这一过程是完全和外界没有沟通的（应该），所以在校园网认证那边看来，就是自己网段下凭空出来了一个新的ipv6。经过长时间的使用，可以确信锐捷校园网认证是通过mac地址识别这种新增的用户的。只要你的mac地址认证了，那么上面过的任何包都可以通过认证。

另外，旁路由的情况也和IPv4类似，IPv4只要加个网关就能上网，IPv6则是加条默认路由就可以了。但是要在防火墙禁掉Openwrt默认发出来的ICMP6-redirect消息。这个消息会让机子重新主动连学校路由器。

#### 在/64地址下如何让内网机器有IPv6

校园网认证的问题暂且不提。IPv6真正的问题是在于路由。校园网只会给里面的设备/64的地址，即不会分配到任何前缀。而其实前缀一般到/64就到底了，从这种意义上说，不能再往下划分任何子网了。但是校园网内确实有路由器的需求。那么解决办法必然是2选1。（第一个方案缺点是非公网地址，外网无法直连，而且配置非常麻烦。这里直接略过。）

1. NAT6
2. relay模式中继，直接拿公网地址。

当你路由器的wan接口有了一个/64的地址之后，那么很自然，这整个/64网段都会必然会被路由到wan接口出去。然而，relay模式把RA消息relay进来了，LAN的机器也是这个网段。这就导致了尴尬的路由问题。假设内网机器想要ping外网ipv6，包走到路由器，路由器从wan发出去，没问题。但是当服务器的响应包从wan回来的时候，到了路由器这里。路由器一看目的地址是wan接口的/64网段，直接把包又丢回wan接口了。。。

#### ndp proxy

NDP，这里主要关注NS和NA，可以想象成是IPv4的ARP。即，我知道包要从这个接口发出去了，但是我要填以太网的mac地址，那我填谁的mac地址呢？解决的是这个问题。这时，需要发送NS请求，询问某个ip地址（网关或者包的目的ip）的机器的mac地址是什么。这时，对面就会发NA响应回来，通告mac地址。

而NDP proxy就是用来解决这种情况的，无论是odhcpd，还是ndppd，都支持在wan监听，发现NS转发进来，如果里面主机响应了，就会增加内核中ipv6的neighbor的缓存表，此时如果开启了对应选项，odhcpd和ndppd都会增加一条/128的专门针对这个主机的路由。从而解决路由问题。

#### ndp proxy的落败

[forum.openwrt.org/t/ipv6-ndp-relay-works-only-on-second-attempt](https://forum.openwrt.org/t/ipv6-ndp-relay-works-only-on-second-attempt/107157/6) （似乎ndppd更好些？然而并不是）

[ndppd/issues/71](https://github.com/DanielAdolfsson/ndppd/issues/71) 不支持转发主机主动发出去的包

[odhcpd 中继模式原理、局限以及解决方案 | Silent Blog (icpz.dev)](https://blog.icpz.dev/articles/notes/odhcpd-relay-mode-discuss/#中继模式的局限性以及可能的解决方法) 正如这篇文章里工作条件里提到的5个步骤，为什么需要5个步骤呢？就是因为现在的ndp proxy有一个非常离谱的缺陷，即只能转发别人发的NS，不能转发路由器自己发出的NS到lan里。这可能是内核的设计问题。内核其实给了什么socket机制，让你只监听NS包，但是这里只能监听到外面来的，不包括自己发出去的。

比如我内网机子有一个`::aaaa`的IP，接入我自己的路由器，然后再接入学校路由器。只有学校路由器发NS请求问`::aaaa`的mac地址是什么的时候，这个包才能被转发到lan内，触发NA响应，然后触发上述的ndp proxy的增加路由机制。

然而经过测试，学校的路由器完全不会发这个NS请求，不知道是缓存了还是什么情况。导致还是最开始的那个问题，缺一条路由。有响应包直接到路由器，但是路由器不会往内网转。非常讽刺的是，其实路由器自身会不断发NS包问内网谁是`::aaaa`，但是经过路由之后，这个NS包还是往wan口发走了。。。也就是说，路由器一直在往wan口找人，却不知道人其实就在自己背后的lan口那边。

[ndppd does receive locally generated neighbor discovery · Issue #69 · DanielAdolfsson/ndppd (github.com)](https://github.com/DanielAdolfsson/ndppd/issues/69) （现在的ndp proxy，包括odhcpd和ndppd，都不支持转发主机主动发出去的包）

但是这个问题的临时解决方案也非常简单，ping一下路由器的公网ip地址即可。ping外面的机子还不管用，必须要ping路由器的公网ipv6。此时路由器就会发现，原来我内网有个邻居，然后触发上面的增加路由机制。

其实现在ipv6已经非常接近正常了，也就是发现没有ipv6的时候要ping一下路由器。在Openwrt上的配置也非常简单：（这里假设interface wan的协议是dhcpv4 client，interface wan6的协议是dhcpv6 client，并且路由器自身ipv6已经正常）

1. Interfaces » WAN » DHCP server » IPv6 Settings里，勾选Designated Master，RA 设置relay，DHCPv6 禁用，NDP proxy设置relay。**勾选learn routes。**
2. Interfaces » LAN » DHCP server » IPv6 Settings里，不能勾选Designated Master，RA 设置relay，DHCPv6 禁用，NDP proxy设置relay。**勾选learn routes。**
3. 注意wan的IPv6 Settings就别动了，全禁用。这边设置relay会导致RA通过wan转发到外面去。

其实可以就这样将就着用了。



### 继续折腾

为了追求完美，每次用ipv6还要ping一下多麻烦啊。仔细想一下问题，其实本质上非常简单，就是差最后一条回到lan的路由。然而就是这追求的一点完美，折腾了好几天都没搞出来。

1. 如果我把整个/64段都路由到lan内

   1. 缺点1：这个网段机器有的在lan内，有的在lan外。加了这个就无法通过ipv6访问路由器外同网段的机器了。

2. 好麻烦啊，明明包就在路由器外边了，就差最后一步连接就架起来了。我最近学了netfilter，nftables，我直接把外面的IPv6包都复制一份进来。

   1. 花了一天拼出了下面的命令

      ```bash
      nft delete table netdev testv6
      nft add table netdev testv6
      nft add chain netdev testv6 na { type filter hook ingress device wan priority -1 \; }
      nft add rule netdev testv6 na ether type ip6 dup to "br-lan" counter return
      nft add rule netdev testv6 na counter return
      ```

   2. 关键的规则就只有一个，即`ether type ip6 dup to "br-lan"`。把所有ipv6包都复制一份进来。但是在lan这边，链路层肯定是不太对的，源mac地址是路由器lan网卡，目的mac地址应该是内网机器的网卡mac。`ether type ip6 ether saddr set xx:xx:xx:xx:xx:xx dup to "br-lan"`虽然改了源MAC地址，但是不知道目的mac地址怎么改了。。。有没有简单的办法知道目的mac地址吗？但是其实这就是NDP做的事情啊。。。如果NDP完美了，上面的方案也就完美了。

      1. 非常神奇的一件事情是，如果你此时正好在wireshark抓包，会发现自己的ipv6是好的。一旦关掉wireshark，ipv6就没了。可能和网卡是否在混杂模式有关。

3. 既然上面NDP proxy的问题是不能抓自己发出的NS包，那我直接用nftables规则把NS包复制一份到LAN口不就行了吗。（话说内核不能直接指定一下，NS包往多个接口发吗，反正目的地址都是广播地址。只要我路由器往LAN发了NS包，自然就知道内网有neighbor了。）（也许是经过路由表路由了）

   1. 搜nftables规则的时候，想在netdev类型里hook egress方向的包。但是发现报错，一搜是内核版本太低了，我的openwrt22.03是5.10内核，然而需要5.17才行。

   2. 但是发现tc （traffic control）可以实现这个功能

      ```bash
      先用nftables把出去的NS包标记上mark 4
      
      tc qdisc add dev wan root handle 1: htb default 30
      tc class add dev wan parent 1: classid 1:30 htb rate 1000mbit ceil 2000mbit burst 15k
      tc class add dev wan parent 1: classid 1:10 htb rate 1000mbit ceil 2000mbit burst 15k
      tc filter add dev wan parent 1:0 prio 1 \
          protocol ipv6 u32 \
          match mark 4 0xff \
          action mirred ingress mirror dev wan
      
      ```

      抓包，确实有NS包复制过来了。然而源mac地址不对。就算有NA响应，路由器会接收吗。。。话说改MAC地址啥的，不就是ndp proxy做的事情吗？应该要改挺多的吧，我直接用这种规则应该应付不过来？

   3. 我能不能直接把这个NS包重新发给自己，从而触发ndp proxy的转发，发到内网？在尝试这一条的时候，我路由器崩了。

4. 路由器你就不能智能一点吗，我tcp包都从你那过了，你还不知道我在lan吗？即在lan口上嗅探包的源IP地址，如果有进入lan口的包，源IP地址在同网段，则增加一条路由。更进一步，如果长时间没有相关请求，可以删除对应的路由。

   1. `libnetfilter_queue`这个库支持用户态处理过滤后的包。更进一步，`libnetfilter_log`支持用户态处理规则记录的包。我可以写一个用户态程序。再进一步，ulogd是一个用了这个库的程序，而且opkg可以直接安装，我直接下载下来配好，然后自己只要写一个python啥的读它的日志，就可以获取到源ipv6了。



#### dhcpv6方案

折腾了这么久，也是非常的累。想到了之前有一篇文章里，那个人也放弃了ndp proxy。

[ndp-proxy-route-ipv6-vpn-addresses](https://quantum5.ca/2019/03/08/ndp-proxy-route-ipv6-vpn-addresses/)

本质上就是要知道lan那边有哪些ip，从而加路由。dhcp不就是用来分发IP的吗，让内网机器直接通过dhcp要地址，然后通过dnsmasq的hook加ndp proxy。

更进一步，可以直接用dhcpv6给内网分发一个小网段，然后直接加静态路由，让这个小网段的包进入lan。

累了，本来还想着说自己搞一个包嗅探，然后加一条路由的程序。这个dhcpv6方案也不错，能用就行。（缺点是，别人一看就知道你这个IP不对劲，肯定是自己配的）

1. 在Openwrt上也可以非常简单地原生配置：

   1. lan的ipv6 setting里，开启dhcpv6 server，RA server。RA里面关闭slaac，开启Manage的flag。
   2. 关闭lan接口Advanced Setting里的IPv6 assignment length然后把wan的/64地址复制一份到那边lan接口。此时客户端就可以分到这个/64段的地址
   3. 注意odhcpd的这个选项：dhcpv6_hostidlength默认值是12 [openwrt.org/docs/techref/odhcpd](https://openwrt.org/docs/techref/odhcpd) 即客户端地址除了低12位，其他会是全零（[https://github.com/openwrt/odhcpd/issues/84](https://github.com/openwrt/odhcpd/issues/84)）。
   4. 再加一个/116的静态路由即可。

2. 自己起一个dnsmasq，可以手选一个IP段。RA的发送还是可以交给odhcpd，毕竟有图形界面。

   [dnsmasq - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/dnsmasq#Test) 
   
   ```
   # dnsmasq -k --conf-file=./dnsmasq.conf --log-facility=- --log-debug
   # ipconfig /release6 以太网
   # ipconfig /renew6 以太网
   
   port=0 # disable dns
   listen-address=::
   
   # enable-ra
   interface=br-lan
   
   dhcp-range=::4321:1,::4321:ffff,constructor:br-lan
   ```

还有没考虑的就是，如果IPv6地址前缀经常变怎么办。。。只能说再看吧。

配置还是得追求简单，可复现，容易维护。太累了自己承担不住。





### 学习资源：

#### nftables

[Quick reference-nftables in 10 minutes - nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes#Icmpv6) 

#### tc traffic control qdisc

[[译\] 《Linux 高级路由与流量控制手册（2012）》第九章：用 tc qdisc 管理 Linux 网络带宽 (arthurchiao.art)](https://arthurchiao.art/blog/lartc-qdisc-zh/)

[流量控制 · GitBook (tonydeng.github.io)](https://tonydeng.github.io/sdn-handbook/linux/tc.html) 

[openwrt.org/docs/guide-user/network/traffic-shaping/packet.scheduler](https://openwrt.org/docs/guide-user/network/traffic-shaping/packet.scheduler#actions) 

#### IPv6

《IPv6技术精要》（美）格拉齐亚尼 著

#### 其他

最后分享一下我的[v2ray透明代理配置](https://forum.openwrt.org/t/solved-usr-share-nftables-d-table-post-not-in-the-right-place/142961/6)



