---
title: Docker与防火墙共存-UFW
date: 2023/5/12 11:11:12
categories:
- Dev
tags:
- Linux
- Networking
---

Docker与防火墙共存-UFW

<!-- more -->

“众所周知”，Docker和UFW（ubuntu自带的防火墙，Uncomplicated Firewall）等一系列防火墙都是不太能共存的。但是最近学校护网，需要打开防火墙。

UFW与Docker不能共存，肯定也有其他人忍不了。最知名的方案大概是[ufw-docker](https://github.com/chaifeng/ufw-docker#%E5%A4%AA%E9%95%BF%E4%B8%8D%E6%83%B3%E8%AF%BB)项目。然而当我看到里面所说的为什么不选用在input链上做那一段的时候，我觉得我想要的就是在input链上防护！折腾由此开始。

最后用了[这个](https://stackoverflow.com/questions/30383845/what-is-the-best-practice-of-docker-ufw-under-ubuntu/58098930#58098930)回答的方案。

[这个github issue](https://github.com/moby/moby/issues/4737#issuecomment-456792819) 涉及的讨论，里面有人提到要改`MANAGE_BUILTINS`

### 操作步骤

1. 修改`/etc/default/ufw`， 把no改成MANAGE_BUILTINS=yes
2. 修改`/etc/ufw/after.rules`（把下面的加到结尾） 注意把eno1改成网卡名字！！
    ```
    # Put Docker behind UFW
    *filter
    :DOCKER-USER - [0:0]
    :ufw-user-input - [0:0]

    -A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT # 只有加上这个才能允许docker的响应包回来。
    -A DOCKER-USER -i eno1 -j ufw-user-input
    -A DOCKER-USER -i eno1 -j DROP
    COMMIT
    ```
3. 依次重启ufw和docker
    ```
    sudo ufw reload
    sudo systemctl restart docker
    ```

### 背后的经验：

如果不增加MANAGE_BUILTINS=yes 刚开始的时候好像没事，但是后面（加规则？和）reload好像会导致ufw直接爆炸。似乎是ufw没有管理DOCKER-USER这个链，但是它引用了ufw-user-input，导致reload过程中关闭ufw的时候无法删掉ufw-user-input链，从而ufw爆炸，解决办法是用sudo iptables -X ufw-user-input手动删掉这个链

如果不重启docker，好像docker的网络直接紊乱了。启动需要转发端口的container的时候报错
```
docker: Error response from daemon: driver failed programming external connectivity on endpoint laughing_northcutt (d38a235dfd7d9a54d19598945713b2d4c47a71eaaef4d9e4e1136d537656673e):  (iptables failed: iptables --wait -t filter -A DOCKER ! -i docker0 -o docker0 -p tcp -d 172.17.0.2 --dport 80 -j ACCEPT: iptables: No chain/target/match by that name.
 (exit status 1)).
 ```

最后一定要注意，每次重启ufw之后都要重启docker，因为MANAGE_BUILTINS=yes之后会把docker自己的ipatables规则都删掉！！需要重启docker增加一下它自己的规则。（经过对比前后iptables -L输出的区别发现）。
