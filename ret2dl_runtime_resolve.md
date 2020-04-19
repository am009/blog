---
title: ret2dl_runtime_resolve
date: 2019/9/8 20:46:25
categories:
- CTF
tags:
- CTF
- rop
---


# ret2dl_runtime_resolve

https://www.jianshu.com/p/e13e1dce095d
https://bbs.pediy.com/thread-227034.htm

<!-- more -->
步骤：
**4. 设置好参数，push下标到栈，跳转到plt的开头（push linkmap; jmp ...）**

**3. 伪造.rel.plt( DT_JMPREL )结构体:
准备一个可写的地址作为r_offset用来填写解析的地址，
填入伪造结构体相对于该节区的下标（r_info）。**
typedef struct
{
  Elf32_Addr    r_offset; 
  Elf32_Word    r_info;
} Elf32_Rel;
00000000 Elf64_Rela      struc ; (sizeof=0x18, align=0x8, copyof_2)
00000000 r_offset        dq ?
00000008 r_info          dq ?
00000010 r_addend        dq ?
00000018 Elf64_Rela      ends
为了计算出这个结构体的内容，准备:
**2. 伪造.dynsym中的结构体，计算r_info**
32位：
typedef struct
{
  Elf32_Word    st_name;
  Elf32_Addr    st_value;
  Elf32_Word    st_size;
  unsigned char st_info; 
  unsigned char st_other;
  Elf32_Section st_shndx;
}Elf32_Sym

其中
st_name：符号名相对.dynstr起始的偏移
st_info：对于导入函数符号此处为0x12

r_info >> 8 == .dynsym下标

64位：
00000000 Elf64_Sym       struc ; (sizeof=0x18, align=0x8, mappedto_1)
00000000 st_name         dd ?                    ; offset (00400318)
00000004 st_info         db ?
00000005 st_other        db ?
00000006 st_shndx        dw ?
00000008 st_value        dq ?                    ; offset (00000000)
00000010 st_size         dq ?
00000018 Elf64_Sym       ends
其中:
st_name 是在.dynstr（STRTAB）中的偏移
st_info == 0x12,
剩下的一般为0
r_info >> 32（8个十六进制） == .dynsym下标

**1. 在某地址处放置字符串，内容是要解析的函数名称，计算相对于.dynstr的下标**


## 经验1
栈迁移后要留足够的栈拓展的空间，给dl_resolve使用，否则栈劫持到bss段开头后，dl_resolve sub esp之后就到代码段了，报错退出

## VERSYM
第二天继续debug，依旧报错不断，还是不能打通。。。不断进行调试。
这里是32位的dl_resolve
突然想到在ctfwiki中隐藏着一句话:
> 注意：
> * 符号版本信息
> 最好使得 ndx = VERSYM[(reloc->r_info) >> 8] 的值为 0，以便于防止找不到的情况。
> * 重定位表项
> r_offset 必须是可写的，因为当解析完函数后，必须把相应函数的地址填入到对应的地址。

所以，我八成是在这里错了，调试dl_resolve停在以下代码
```
In file: /home/wjk/glibc32/glibc-2.29/elf/dl-runtime.c
   88       if (l->l_info[VERSYMIDX (DT_VERSYM)] != NULL)
   89 	{
   90 	  const ElfW(Half) *vernum =
   91 	    (const void *) D_PTR (l, l_info[VERSYMIDX (DT_VERSYM)]);
   92 	  ElfW(Half) ndx = vernum[ELFW(R_SYM) (reloc->r_info)] & 0x7fff;
 ► 93 	  version = &l->l_versions[ndx];
   94 	  if (version->hash == 0)
   95 	    version = NULL;
   96 	}
   97 
   98       /* We need to keep the scope around so do some locking.  This is
```
开始分析上下文汇编
```
   0xf7f4db55 <_dl_fixup+101>    mov    edx, dword ptr [edx + 4]
   0xf7f4db58 <_dl_fixup+104>    movzx  edx, word ptr [edx + esi*2]
   0xf7f4db5c <_dl_fixup+108>    and    edx, 0x7fff
 ► 0xf7f4db62 <_dl_fixup+114>    shl    edx, 4
   0xf7f4db65 <_dl_fixup+117>    add    edx, dword ptr [eax + 0x174]
   0xf7f4db6b <_dl_fixup+123>    mov    ebx, dword ptr [edx + 4]
   0xf7f4db6e <_dl_fixup+126>    test   ebx, ebx
   0xf7f4db70 <_dl_fixup+128>    mov    ebx, 0
   0xf7f4db75 <_dl_fixup+133>    cmove  edx, ebx
```
第一句把VERSYM地址(0x080482e4)放到了edx
EDX  0x80482e4 ◂— 0x20000
而ESI  0x215

是对应符号的下标(r_info >> 8)(r_info)
也就是说\[edx + esi*2\]就是计算VERSYM\[(reloc->r_info) >> 8\]的方式吗？所以每个符号信息占2bit？

对比调试发现，ctfwiki的实例脚本就是在第二句，取到了0赋值给edx（esi = 0x269），而我的脚本则是取到了离谱的0xffff !!?为什么/(ㄒoㄒ)/~~

而下面的shl edx, 4和add的组合使用又像是把它作为结构体大小为0x10的下标使用，不过eax+0x174 是存的0xf7ecf3f0
```
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
 0x8048000  0x8049000 r-xp     1000 0      /media/wjk/ha/ctf/ctfwikipwn/stackoverflow/ret2dlresolve/XDCTF-2015/main
 0x8049000  0x804a000 r--p     1000 0      /media/wjk/ha/ctf/ctfwikipwn/stackoverflow/ret2dlresolve/XDCTF-2015/main
 0x804a000  0x804b000 rw-p     1000 1000   /media/wjk/ha/ctf/ctfwikipwn/stackoverflow/ret2dlresolve/XDCTF-2015/main
0xf7cd5000 0xf7cf2000 r--p    1d000 0      /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7cf2000 0xf7e42000 r-xp   150000 1d000  /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7e42000 0xf7eae000 r--p    6c000 16d000 /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7eae000 0xf7eaf000 ---p     1000 1d9000 /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7eaf000 0xf7eb1000 r--p     2000 1d9000 /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7eb1000 0xf7eb3000 rw-p     2000 1db000 /usr/lib/i386-linux-gnu/libc-2.29.so
0xf7eb3000 0xf7eb5000 rw-p     2000 0      
0xf7ecf000 0xf7ed1000 rw-p     2000 0      
0xf7ed1000 0xf7ed4000 r--p     3000 0      [vvar]
0xf7ed4000 0xf7ed5000 r-xp     1000 0      [vdso]
0xf7ed5000 0xf7ed6000 r--p     1000 0      /usr/lib/i386-linux-gnu/ld-2.29.so
0xf7ed6000 0xf7ef2000 r-xp    1c000 1000   /usr/lib/i386-linux-gnu/ld-2.29.so
0xf7ef2000 0xf7efc000 r--p     a000 1d000  /usr/lib/i386-linux-gnu/ld-2.29.so
0xf7efd000 0xf7efe000 r--p     1000 27000  /usr/lib/i386-linux-gnu/ld-2.29.so
0xf7efe000 0xf7eff000 rw-p     1000 28000  /usr/lib/i386-linux-gnu/ld-2.29.so
0xffa36000 0xffd69000 rw-p   333000 0      [stack]
```
这在什么区域？什么也没标？？！

不管那么多了，我还是看好自己的下标吧，0x080482e4+2\*0x215=0x804870e
居然在.eh_frame...
于是把下标(r_info >> 8)增大2试试
成功调用write！！！
```
.eh_frame:0804870B                 db    0
.eh_frame:0804870C                 db 0B4h
.eh_frame:0804870D                 db 0FEh
.eh_frame:0804870E                 db 0FFh
.eh_frame:0804870F                 db 0FFh
.eh_frame:08048710                 db  5Dh ; ]
.eh_frame:08048711                 db    0
.eh_frame:08048712                 db    0
.eh_frame:08048713                 db    0
.eh_frame:08048714                 db    0
.eh_frame:08048715                 db  41h ; A
.eh_frame:08048716                 db  0Eh
```














