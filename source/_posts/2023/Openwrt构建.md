---
title: Openwrt构建
date: 2024/01/11 11:11:12
categories:
- Dev
tags:
- Networking
---

Openwrt构建

<!-- more -->

### 编译环境搭建

一些[常用命令](https://hackmd.io/@gtknw/building_guide)

- 跟着[install-buildsystem](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem)安装一堆包
- 跟着[use-buildsystem](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem)克隆源码
- 跟着[using_official_build_config](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem#using_official_build_config)构建一个路由器能用的镜像。我手头的路由器是AX6S，ROM的下载链接是：`http://downloads.openwrt.org/releases/23.05.0/targets/mediatek/mt7622/openwrt-23.05.0-mediatek-mt7622-xiaomi_redmi-router-ax6s-squashfs-factory.bin`，所以我就执行：`wget https://mirror-01.infra.openwrt.org/releases/23.05.0/targets/mediatek/mt7622/config.buildinfo -O .config` 和 `wget https://mirror-01.infra.openwrt.org/releases/23.05.0/targets/mediatek/mt7622/feeds.buildinfo -O feeds.conf`
- 构建GRE、netifd包：`make package/network/config/netifd/compile V=s` （加上了`V=s`可以看到使用的相关命令）

### 包编译的细节

- 首先源码的压缩包，会被下载：`openwrt/dl/netifd-2023-09-19-7a58b995.tar.xz`
- 然后被解压：`openwrt/build_dir/target-aarch64_cortex-a53_musl/netifd-2023-09-19-7a58b995`
- 在上面文件夹里进行编译
- 被安装：`openwrt/build_dir/target-aarch64_cortex-a53_musl/netifd-2023-09-19-7a58b995/ipkg-install/usr/sbin/netifd`
- 打包

### 单个包的开发

- [在feed中增加`src-link`](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem#creating_a_local_feed)，关联本地源码。

### 给核心仓库提交代码

- [Submitting patches](https://openwrt.org/submitting-patches)页面，提到了核心包可以通过Github PR或者邮件列表提交。
- [这个Patch](https://lists.openwrt.org/pipermail/openwrt-devel/2024-January/041977.html)先出现在邮件列表，然后[commit](https://git.openwrt.org/?p=project/netifd.git;a=commit;h=4219e99eeec7514657f5838eb4b4b5eb28ee1271)才出现在netifd的列表里，说明他们还是在用mailing list的，可以考虑直接写patch交patch。
