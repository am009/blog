---
title: CSU-gadget总结
date: 2020/1/15 20:46:25
categories:
- CTF
tags:
- libc
- CTF
---


# CSU-gadget总结
<!-- more -->

csu这个gadget有点强啊
它在brop里也叫brop gadget

```
0x4005c0 <__libc_csu_init>:	push   r15
0x4005c2 <__libc_csu_init+2>:	push   r14
0x4005c4 <__libc_csu_init+4>:	mov    r15d,edi
0x4005c7 <__libc_csu_init+7>:	push   r13
0x4005c9 <__libc_csu_init+9>:	push   r12
0x4005cb <__libc_csu_init+11>:	lea    r12,[rip+0x20083e]        # 0x600e10
0x4005d2 <__libc_csu_init+18>:	push   rbp
0x4005d3 <__libc_csu_init+19>:	lea    rbp,[rip+0x20083e]        # 0x600e18
0x4005da <__libc_csu_init+26>:	push   rbx
0x4005db <__libc_csu_init+27>:	mov    r14,rsi
0x4005de <__libc_csu_init+30>:	mov    r13,rdx
0x4005e1 <__libc_csu_init+33>:	sub    rbp,r12
0x4005e4 <__libc_csu_init+36>:	sub    rsp,0x8
0x4005e8 <__libc_csu_init+40>:	sar    rbp,0x3
0x4005ec <__libc_csu_init+44>:	call   0x400400 <_init>
0x4005f1 <__libc_csu_init+49>:	test   rbp,rbp
0x4005f4 <__libc_csu_init+52>:	je     0x400616 <__libc_csu_init+86>
0x4005f6 <__libc_csu_init+54>:	xor    ebx,ebx
0x4005f8 <__libc_csu_init+56>:	nop    DWORD PTR [rax+rax*1+0x0]
！！！csu-front
0x400600 <__libc_csu_init+64>:	mov    rdx,r13
0x400603 <__libc_csu_init+67>:	mov    rsi,r14
0x400606 <__libc_csu_init+70>:	mov    edi,r15d
0x400609 <__libc_csu_init+73>:	call   QWORD PTR [r12+rbx*8]
0x40060d <__libc_csu_init+77>:	add    rbx,0x1
0x400611 <__libc_csu_init+81>:	cmp    rbx,rbp
0x400614 <__libc_csu_init+84>:	jne    0x400600 <__libc_csu_init+64>
0x400616 <__libc_csu_init+86>:	add    rsp,0x8
！！！csu-rear
0x40061a <__libc_csu_init+90>:	pop    rbx
0x40061b <__libc_csu_init+91>:	pop    rbp
0x40061c <__libc_csu_init+92>:	pop    r12
0x40061e <__libc_csu_init+94>:	pop    r13
0x400620 <__libc_csu_init+96>:	pop    r14
0x400622 <__libc_csu_init+98>:	pop    r15
0x400624 <__libc_csu_init+100>:	ret   
```

0x40061a 0 <__libc_csu_init+90>:	pop    rbx
0x40061b 1 <__libc_csu_init+91>:	pop    rbp
0x40061c 2 <__libc_csu_init+92>:	pop    r12
0x40061e 4 <__libc_csu_init+94>:	pop    r13
0x400620 6 <__libc_csu_init+96>:	pop    r14  0x400620
0x400622 8 <__libc_csu_init+98>:	pop    r15
0x400624 10 <__libc_csu_init+100>:	ret   

而在pop r14的地方断开，有着 pop rsi; pop r15; ret
在r15的地方断开，有着pop rdi; ret

一定要记住。
另外brop中提到了pop rdx的gadget几乎没有但是可以用strcmp来控制

另外它的来历也大不一般。它是负责在启动main函数之前，去调用init_array的每一个函数的。
此外，基本上只有这里有pop ret的gadget了

## 16字节对齐：
这里采取典型的ctf-wiki上使用的csu-rear再接着csu-front进行研究
对于16字节对齐，我把8字节和8字节进行配对。
call是特殊的push，retn是特殊的pop
平时在程序里面，call和之后的push ebp配对
leave 和return 配对

现在开始rop，程序开始leave，ret，跳转到csu-rear
接下来是pop rbx rbp r12 r13 r14 r15偶数个pop
接下来是又是ret，这次没有leave，导致了缺了8字节



接下来是csu-front
三个move后直接call，这个时候有push和它配对，然而之前的ret没有leave，所以不太好，栈偏了

总结：csu-rear会造成栈偏移8字节，因为leave没有return配对，但是调用一次front和一次rear之后，其实调用了两次rear，抵消了，导致了栈恢复。。。
然而在call的时候，栈的状态是不对的

解决方法： 开头的时候调用一次csu-rear，这样整个栈就偏了8字节，调用函数的时候就还是对齐的

我们平时进行rop的时候，直接使用gadget是对齐的。
因为rop之前，leave（另外导致了rbp变化）和ret配对，call和函数开头的push ebp配对


https://wooyun.js.org/drops/%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5%E5%AD%A6ROP%E4%B9%8Bgadgets%E5%92%8C2free%E7%AF%87.html