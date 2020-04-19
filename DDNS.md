---
title: DDNS
date: 2019/11/25 20:46:25
categories:
- linux
tags:
- linux
- raspberrypi
---

# DDNS

之前想做DDNS都做不成,最近突然发现dns注册商都有API

1. 我可以使用一些获取ip的公网api,比如
http://v4.ipv6-test.com/api/myip.php
http://v6.ipv6-test.com/api/myip.php
http://v4v6.ipv6-test.com/api/myip.php

2. 我可以使用ipconfig,ifconfig之类的获取ip地址.
python获取ip地址
https://stackoverflow.com/questions/24196932/how-can-i-get-the-ip-address-from-nic-in-python
学习使用netifaces模块
https://pypi.org/project/netifaces/
总结:
```python
import netifaces as ni
import ipaddress
def ipv4():
    '''
    return the non loopback ipv4 address of the computer
    sample output: ['10.122.122.122']
    '''
    its = ni.interfaces()
    ips = [ni.ifaddresses(i)[ni.AF_INET][0]['addr'] for i in its]
    ips = [i for i in ips if not ipaddress.IPv4Address(i).is_loopback]
    return ips

def ipv6():
    '''
    return the non loopback ipv4 address of the computer
    sample output: ['2001:da8:215:8f01::1']
    AF_INET6
    '''
    its = ni.interfaces()
    ips = [ni.ifaddresses(i)[ni.AF_INET6][0]['addr'] for i in its]
    ips = [i for i in ips if not ipaddress.IPv6Address(i).is_loopback]
    return ips

```

1. 我可以把ip都传到服务器上去,服务器统一管理
2. 我可以每台电脑单独修改解析记录

