---
title: X11转发-从docker到WSL
date: 2023/10/28 11:11:12
categories:
- Read
tags:
- Networking
---

X11转发-从docker到WSL

<!-- more -->

随着WSL-g的普及，只需要在WSL2中去SSH连接远程服务器时带上-X或者-Y，就能够把服务器上的图形界面程序转到Windows本机，这在过去需要本机是Linux电脑。

如果远程服务器上装了docker，想在docker里再跑图形界面程序则更加复杂。需要从docker转到服务器，然后再通过SSH转到本地WSL。为了更好地使用这些高级的操作，有必要学习相关的原理。

### SSH X11 转发

XSOCK环境变量负责告诉图形界面程序，怎么去连接X11服务器，从而显示图像。在通常的ubuntu desktop系统中，往往图形服务器使用的是`/tmp/.X11-unix`。参考[这里](https://unix.stackexchange.com/questions/196677/what-is-tmp-x11-unix)。

SSH连接服务器如果带上了-X，则会设置DISPLAY和XAUTHORITY，DISPLAY通常指定为`hostname:10.0`。此时用的就是普通的tcp的端口，即`6000+10=6010`。

此外，一般这个端口只允许localhost的连接，由sshd_config里的`X11UseLocalhost no`控制。

### XAUTH 和 `.Xauthority`

TODO。不过根据[这里](https://dzone.com/articles/docker-x11-client-via-ssh)，把`.Xauthority`文件挂载进去就好了:

```
--volume="$HOME/.Xauthority:/root/.Xauthority:rw"
```

### docker X11 + SSH X11：要挂载`/tmp/.X11-unix`吗？

场景是，笔记本使用windows，连接服务器，服务器上在docker里运行图形界面程序，直接转发到笔记本显示。

在[这里](https://github.com/GrammaTech/retypd-ghidra-plugin)介绍的转发方法使用了下面的挂载：

```
xhost +si:localuser:root
docker run -it --privileged \
    --network=host \
    -e DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    retypd-image \
    bash
```

- 挂载了`/tmp/.X11-unix`目录，这意味着假设x11转发走的是这里面的unix socks（一般本机需要是linux系统才会用这个）
- `--network=host`使用本机网络，`-e DISPLAY`把当前的DISPLAY变量传进去。

但是我们是ssh的x11转发，所以和`/tmp/.X11-unix`没关系，可以放心地去掉这个挂载，最后我们得到下面几个flag：

```
    --network=host \
    -e DISPLAY \
    --volume="$HOME/.Xauthority:/root/.Xauthority:rw" \
```

