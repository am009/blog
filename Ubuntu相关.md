---
title: Ubuntu相关
date: 2019/7/31 20:46:25
categories:
- linux
tags:
- linux
- ubuntu
---

# Ubuntu


<!-- more -->

apt-get就不多说了
apt-cache search 可以搜索软件包
apt-cache depends 搜索依赖

## bash_history无限大
修改~/.bashrc HISTSIZE 和 HISTFILESIZE 为 -1

## gnome远程桌面使用vnc viewer

gnome的vnc软件vino在加密方式上和vnc viewer有什么不协调，就是不能连上。。。
搜索无果，只好关闭加密
gsettings是一个管理gnome桌面设置的cmd工具，它还有图形界面编辑器gconf-editor，这个也可以达到一样的效果。
gsettings set org.gnome.Vino require-encription false