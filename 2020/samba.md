---
title: 设置最新版本的windows10能够访问的samba服务器
date: 2020/6/1 11:11:11
categories:
- Dev
tags:
- Windows
---

# 设置最新版本的windows10能够访问的samba服务器

目标：需要有只读的公共访问和可读可写的非公共访问

注意的地方是, 第一次访问可能会询问用户名密码, 不代表guest配置失败, 乱输用户名即可通过`map to guest = bad user` 作为guest。坑惨我了

<!-- more -->

现在回头看，总感觉微软只关注了那种公司内使用了域控服务器管理了大量主机，从而可以相互认证的情况，而不关心我们这种笔记本用户的“孤岛”情况，导致`"默认"`并不是无密码，而是当前用户的账号密码。无密码访问挺不容易的。

## 关键配置1 加密和签名
```
	server signing = mandatory
#	smb encrypt = mandatory
```
查看日志发现, 新版本的win10(还是samba服务器??)对没有加密也没有签名的连接会拒绝. 所以需要这两个选项.

加密必须要用户名和密码, 因为加密的会话密钥就是和用户名关联的. 因此为了guest用户, 需要注释掉加密的选项.

来自[How to enable SAMBA encryption and do not require user authentication
](https://serverfault.com/questions/874423/how-to-enable-samba-encryption-and-do-not-require-user-authentication)

## 关键配置2
```
#	min protocol = SMB2
```
自从win10开始, 默认使用的就是SMB3_11了, 在 `启用或关闭windows功能` 里开启SMB1/CIFS, 访问时就可能会使用SMB2. 这里调试的时候可以考虑强制改成SMB3. 发现关键问题之后为了兼容性注释掉了

## 关键配置3
```
	guest account = guest
	null passwords = yes
```
默认是`guest account = nobody`. #TODO
第二个参数 `null passwords` 不加上, 使用空密码登录的时候就会被拒绝.

## 关键配置4 ntlm auth
```
	ntlm auth = ntlmv1-permitted
	lanman auth = yes
	raw NTLMv2 auth = yes
```
win10可能会使用ntlmv1, 而经过永恒之蓝事件之后samba默认只接受ntlmv2了.
关键的只是第一条, 后面的两条是逛的时候发现的, 加了可以增加兼容性.

[Samba and ntlm for Windows clients](https://bgstack15.wordpress.com/2017/10/01/samba-and-ntlm-for-windows-clients/)

[samba config ntlmauth](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#NTLMAUTH)

## 关键配置5 passdb backend
```
	passdb backend = smbpasswd:/etc/samba/smbpasswd
	smb passwd file = /etc/samba/smbpasswd
```
新版本默认使用的不是smbpasswd, 默认的数据库位置更不是 `/etc/samba/smbpasswd`. 新版本似乎用的是 `pdbedit`? 

指定数据库文件位置似乎是用第一行的方式了, 第二行似乎没有效果了?

## 关键配置6 force user
```
	force user = pi
```
强制用户了之后上传的文件的所有者就都是一样的了

## 总体配置:
```
[global]
    map to guest = Bad User
	server signing = mandatory
#	smb encrypt = mandatory
#	min protocol = SMB2
	passdb backend = smbpasswd:/etc/samba/smbpasswd
	smb passwd file = /etc/samba/smbpasswd
	guest account = guest
	null passwords = yes
	security=user
	ntlm auth = ntlmv1-permitted
	lanman auth = yes
	raw NTLMv2 auth = yes

[ro]
        # This share allows anonymous (guest) access
        # without authentication!
        path = /home/pi/
#	force user = pi
        read only = yes
        guest ok = yes
#        guest only = yes

[rw]
	path = /home/pi/
	read only = no
	valid users = pi
	force user = pi
```

## debug方法

windows 清除登录密码首先要凭据管理器删除
接着我任务管理器关闭explorer再启动, 没有用, 只有重启

debug samba 的方法
```
sudo service smbd stop
sudo smbd -F -S -d=10
```
此时再连接, 就可以看到debug信息了. -d指定的loglevel从1到10.
`-d=5`的时候的log就已经很多了, `-d=3` 的时候log不是很多.平时一般先使用 `-d=3`
