---
title: 天枢的新手入门指南 Q&A
date: 2019/7/31 20:46:25
categories:
- Hack
tags:
- CTF
---

这篇文章是2019年7月31日从天枢新人群里复制来的。增加自己博客的文章数

# 天枢的新手入门指南 Q&A

## 什么是天枢

### 天枢战队

<!-- more -->

> ​	 天枢战队是来自北京邮电大学（BUPT）的一群小伙伴组成的安全团队。队名“天枢”是北斗七星的第一颗星，它代表了聪慧和才能。核心团队有十五人左右，活跃在国内外大大小小的赛事上。大家聚在一起，乐于学习、研究和交流各个方向的安全技术，提高北邮信息安全氛围。
>
> ​	队员们专精的技能千奇百怪：手捏网线就能发包，口算sha256比2080ti还快，盯着字节码即可逆向操作系统，双击Chrome就能v8逃逸，通过人眼扫描面部即可微信添加好友等。

### 天枢社团

> ​	天枢社团的主要作用就是向天枢战队输送新生力量，提高北邮的民间信息安全能力与氛围。定期组织CTF交流，提高自身的信息安全技能

### 如何加入天枢战队
天枢战队在每年5月底6月初会举办`TSCTF`邀请赛，在线上赛取得突出成绩，或者在其他比赛中有优异成绩即可进入天枢战队

### 天枢社团的预想组织架构
- 中心组
    - 活动（负责相关活动的宣传，组织，评定，场地，以及后期的报销等事宜）//这里可能可以细分？
    - 技术（负责相关活动的技术维护）
- XX战队（负责参加各种比赛，包括但不限于国赛，XCTF联赛等，人数大概维持在20人-30人左右）

### 天枢社团预想学习模式
1. 在开学初进行战队的初级选拔(大概招30人吧，我也不清楚)，主要选拔一些有基础，或者说感兴趣并想坚持的人。
2. 开学之后会进行1-2个月左右的学习，然后进行选拔，抽取前20(概率还是挺大的)
3. 每周面向全校（主要是战队）进行沙龙，不同方向和领域可以分开进行活动。
4. 每月进行一次战队的交流，比赛（评定），分享学习经验。提高合作水平
5. 战队会进行定期考核，当无法完成相应任务，会从战队中移除。

##  什么是CTF

> ​	CTF（Capture The Flag）中文一般译作夺旗赛，在网络安全领域中指的是网络安全技术人员之间进行技术竞技的一种比赛形式。CTF起源于1996年DEFCON全球黑客大会，以代替之前黑客们通过互相发起真实攻击进行技术比拼的方式。发展至今，已经成为全球范围网络安全圈流行的竞赛形式，2013年全球举办了超过五十场国际性CTF赛事。而DEFCON作为CTF赛制的发源地，DEFCON CTF也成为了目前全球最高技术水平和影响力的CTF竞赛，类似于CTF赛场中的“世界杯” 。(百度百科)

CTF的几个方向 Web（网络安全）,Pwn（二进制安全）,Re（逆向工程）,Crypto（密码学）,Misc（流量分析，图片隐写，取证，等其他方向）

## 方向介绍，入门书籍及练习网站

### Web

> 网络安全，及负责各种数据库，网页，等网络产品的漏洞挖掘及其利用

**书单**

| 书名                           | 网购链接（没有打广告）                                       | 电子书链接                                               |
| ------------------------------ | ------------------------------------------------------------ | -------------------------------------------------------- |
| 白帽子讲Web安全                | [白帽子讲Web安全](https://detail.tmall.com/item.htm?spm=a230r.1.14.16.6c384122VZ3uBX&id=596872157850&cm_id=140105335569ed55e27b&abbucket=3) | 群文件自取                                               |
| 代码审计 企业级Web代码安全架构 | [代码审计](https://detail.tmall.com/item.htm?spm=a230r.1.14.9.396746e1Anlg9X&id=594217706164&cm_id=140105335569ed55e27b&abbucket=3) | [下载](https://u1475340.ctfile.com/fs/1475340-228355356) |
| Sndav弱鸡的博客                | [博客](http://blog.boyblog.club)                             |                                                          |
| 离别歌的博客                   | [博客](https://www.leavesongs.com/)                          |                                                          |
| 郁离歌丶的博客                 | [郁离歌丶的博客](http://yulige.top/)                         |                                                          |
| SecWiki                        | [SecWiki](https://sec-wiki.com/)                             |                                                          |
| WooYun镜像站                   | [WooYun镜像站](http://www.anquan.us/)                        |                                                          |
| CTF-Wiki                       | [Wiki](https://ctf-wiki.github.io/ctf-wiki/)                 |                                                          |

**练习网站**

| 网站               | 地址                                                      | 难度      |
| ------------------ | --------------------------------------------------------- | --------- |
| 攻防世界           | [攻防世界](http://adworld.xctf.org.cn)                    | 初级-困难 |
| 北京联合大学OJ     | [BUUOJ](https://buuoj.cn/)                                | 中等      |
| Github CTFTraining | [CTFTraining](https://github.com/CTFTraining/CTFTraining) | 中级-困难 |





### Pwn

> 二进制安全，负责各种二进制程序（ELF，EXE等）的漏洞挖掘及其利用

**书单**

| 书名 | 网购链接 | 电子书地址 |
| ---- | ---- | ---- |
| 0day安全:软件漏洞分析技术 | [0day安全:软件漏洞分析技术](https://detail.tmall.com/item.htm?spm=a230r.1.14.33.47641d36xHPMdd&id=595379481437&ns=1&abbucket=3) | 群内自取 |
| 汇编语言 | [汇编语言](https://item.jd.com/12259774.html) |
|程序员的自我修养（装载，链接与库）|自己淘宝吧|[百度网盘，提取码：73pe](https://pan.baidu.com/s/1cALpx_D_9CR9hWWM9rIMwQ)|
|深入理解计算机系统|同上|[百度网盘，提取码：yx0s](https://pan.baidu.com/share/init?surl=gtB8fEUUtFj8blwJnajICQ)|
|glibc内存管理ptmalloc2源代码分析|同上|[百度网盘，提取码：su8n](https://pan.baidu.com/s/1-0odrFdV0Dn7xgehicuz0A)|
|xxrw的blog|[博客](https://xiaoxiaorenwu.top)||
|天枢-p4nda的blog|[博客](http://p4nda.top/)||
|天枢-YM的blog|[博客](https://e3pem.github.io)||
|天枢-17的blog|[博客](https://sunichi.github.io/)||
|r3kapig-swing的blog|[博客](https://bestwing.me/)||
|VidarTeam-Veritas501的blog|[博客](https://veritas501.space/)||
**资料**
| 网站 | 地址 |
| ---- | ---- |
| CTF-Wiki | [Wiki](https://ctf-wiki.github.io/ctf-wiki/) |
| CTF-ALL-In-One | [CTF-ALL-In-One](https://github.com/firmianay/CTF-All-In-One/) |
| Shellcode网站1 | [Shell-Storm](https://shell-storm.org/) |
|Shellcode网站2|[shellcode](https://www.exploit-db.com/shellcodes)|
|北京邮电大学瑶光战队学习资料|[北邮瑶光](https://github.com/xiaoxiaorenwu/-)|
|i春秋的pwn基础教程|[i春秋搜索pwn入门](https://bbs.ichunqiu.com/search.php?mod=portal&searchid=117&searchsubmit=yes&kw=pwn%E5%85%A5%E9%97%A8)|
|看学知识库|[看雪知识库](https://www.kanxue.com/chm-search-pwn.htm)|
|libc搜索|[libc搜索](http://libcdb.com/)|

**刷题网站**
| 网站 | 地址 | 难度 |
| ---- | ---- | ---- |
| 攻防世界 | [攻防世界](http://adworld.xctf.org.cn) | 初级-困难 |
| 北京联合大学OJ | [BUUOJ](https://buuoj.cn/) | 中等 |
| Github CTFTraining | [CTFTraining](https://github.com/CTFTraining/CTFTraining) | 中级-困难 |
| PwnableKr | [PwnableKr](https://pwnable.kr) | 初级-中级 |
| PwnableTw  | [PwnableTw](https://pwnable.tw/) | 中级-困难 | 
| Jarvisoj | [Jarvisoj](https://Jarvisoj.com) | 初级-中级|
| CTFWP | [CTFWP](http://www.ctfwp.com) | 中级 |


### Re

> 逆向工程，负责逆向程序算法，破解程序限制

**书单**

| 书名 | 网购链接 | 电子书地址 |
| ---- | ---- | ---- |
|   Re4b(Reverse Engineer For beginner)   |    [Re4b](https://item.jd.com/12166962.html)  |   群内自取   |
|   加密与解密  |   [加密与解密](https://detail.tmall.com/item.htm?spm=a230r.1.14.189.1d8238f5fF5n7B&id=580607194609&ns=1&abbucket=3)   |   [百度云](https://pan.baidu.com/s/18PhiF_STy4413w4rlfZKfQ) 提取码: hpdb   |
| CTF-Wiki                       | [Wiki](https://ctf-wiki.github.io/ctf-wiki/)                 |                                                          |
**练习网站**

| 网站               | 地址                                                      | 难度      |
| ------------------ | --------------------------------------------------------- | --------- |
| 攻防世界           | [攻防世界](http://adworld.xctf.org.cn)                    | 初级-困难 |
| 北京联合大学OJ     | [BUUOJ](https://buuoj.cn/)                                | 中等      |
| Github CTFTraining | [CTFTraining](https://github.com/CTFTraining/CTFTraining) | 中级-困难 |




### Crypto

> 密码学，负责通过密码以及数学知识，破解相应密码

**书单**

| 书名 | 淘宝链接 | 电子书地址 |
| ---- | ---- | ---- |
|    现代密码学  |      |      |
| CTF-Wiki                       | [Wiki](https://ctf-wiki.github.io/ctf-wiki/)                 |                                                          |

**练习网站**

| 网站               | 地址                                                      | 难度      |
| ------------------ | --------------------------------------------------------- | --------- |
| 攻防世界           | [攻防世界](http://adworld.xctf.org.cn)                    | 初级-困难 |
| 北京联合大学OJ     | [BUUOJ](https://buuoj.cn/)                                | 中等      |
| Github CTFTraining | [CTFTraining](https://github.com/CTFTraining/CTFTraining) | 中级-困难 |





### Misc

> 杂项，负责各种其他安全相关，比如取证，隐写，区块链等

**书单**

| 书名 | 网购链接 | 电子书地址 |
| ---- | ---- | ---- |
| CTF-Wiki                       | [Wiki](https://ctf-wiki.github.io/ctf-wiki/)                 |                                                          |

**练习网站**

| 网站               | 地址                                                      | 难度      |
| ------------------ | --------------------------------------------------------- | --------- |
| 攻防世界           | [攻防世界](http://adworld.xctf.org.cn)                    | 初级-困难 |
| 北京联合大学OJ     | [BUUOJ](https://buuoj.cn/)                                | 中等      |
| Github CTFTraining | [CTFTraining](https://github.com/CTFTraining/CTFTraining) | 中级-困难 |



