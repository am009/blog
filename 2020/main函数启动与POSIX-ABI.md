---
title: main函数启动与POSIX-ABI
date: 2020/1/29 20:46:25
categories:
- Hack
tags:
- Linux
- CTF
---


# main函数启动与POSIX-ABI

<!-- more -->

https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-4.html
这篇文章不错

https://embeddedartistry.com/blog/2019/04/08/a-general-overview-of-what-happens-before-main/
这篇文章的拓展阅读不少好东西：
https://lwn.net/Articles/631631/
## 初始时的栈布局
https://luomuxiaoxiao.com/?p=516
这篇文章也不错

> 3.2.1 首先，_start是如何启动的？
> 当你执行一个程序的时候，shell或者GUI会调用execve()，它会执行linux系统调用execve()。如果你想了解关于execve()函数，你可以简单的在shell中输入man execve。这些帮助来自于man手册（包含了所有系统调用）的第二节。简而言之，系统会为你设置栈，并且将argc，argv和envp压入栈中。文件描述符0，1和2（stdin, stdout和stderr）保留shell之前的设置。加载器会帮你完成重定位，调用你设置的预初始化函数。当所有搞定之后，控制权会传递给_start()，下面是使用objdump -d prog1输出的_start函数的内容：

所以程序的运行过程就是，系统把elf的规定好的几个段加载进去，然后从_start(entry_point)运行。但是这时候，难道栈上什么都没有吗？
为了探究在进入entry_point时候的栈上的数据，看x86-64-ps-ABI.pdf。
在Low Level Interface > Process Initialization > Initial Stack and Register State这里的图3-9就表示了初始化时的栈布局：
这里先提一下寄存器的布局，除了rsp和rdx其他的寄存器的内容都是未定义的。其中rbp被点明需要清零，rdx是需要注册到退出前的函数的（application should register it with atexit）(BA_OS).这里观察到是_dl_fini。r13寄存器的值观察到和rsp一样，r12和rip一样。r9指向0x400000,rsi指向一个ld.so的数据段下方的无名地址。rax我还以为是execve的系统调用号，很可惜不是，而是0x1c。由于最先接管程序的反而是ld.so，所以这里的数据是什么在于它最后做了什么。而且栈的低地址方向上还有不少各种各样的数据，估计也是它的。有大概0xd10 3344字节的脏数据。。。
![stack layout](../imgs/psABIstack.png)
总之栈初始时是有东西的，而且还不少！随便找一个64位的程序用gdb打开，start自动停在入口点，就可以看到：
从低地址到高地址，首先rsp指向的是 argument count，接下来是对应数量的参数指针，接着是参数和环境变量之间的8字节空白分隔，
接下来是环境变量的指针数组，libc的全局变量environ就是指向这里。key和value没有分开，在同一个字符串里面用等于号连接。这样每一个指针就指向一个带等于号的字符串。然后又是一个8字节的0分隔。（可想而知如果程序简单，main函数的栈只有几十字节大小，环境变量数组很容易就被溢出了，导致调用system失败。。。）
接下来是一些不明意义的数据，叫做Auxiliary vector entries，每个16字节，前8字节是类型，后8字节是内容。这里接下来好像是0x18字节的分隔？
https://lwn.net/Articles/519085/ 可以通过getauxval()这个libc的函数调用获得
我对照着表把这次运行的flag都标注了一下
```
24:0120│          0x7fffffffe380 ◂— 0x21 /* '!' */ AT_SYSINFO_EHDR？
25:0128│          0x7fffffffe388 —▸ 0x7ffff7ffa000 ◂— jg     0x7ffff7ffa047 # vdso 的地址。 link：https://www.jianshu.com/p/071358f497ea
26:0130│          0x7fffffffe390 ◂— 0x10 AT_HWCAP
27:0138│          0x7fffffffe398 ◂— 0x78bfbff  an bitmask of CPU features. It mask to the value returned by CPUID 1.EDX.
28:0140│          0x7fffffffe3a0 ◂— 0x6 AT_PAGESZ
29:0148│          0x7fffffffe3a8 ◂— 0x1000 in bytes. 这就是为什么加载时最后三位都是0吧
2a:0150│          0x7fffffffe3b0 ◂— 0x11 AT_CLKTCK
2b:0158│          0x7fffffffe3b8 ◂— 0x64 /* 'd' */  contains the frequency at which times() increments.
2c:0160│          0x7fffffffe3c0 ◂— 0x3 AT_PHDR
2d:0168│          0x7fffffffe3c8 —▸ 0x400040 ◂— 0x500000006 给脚本文件开头的#!/bin/bash之类的用的， tells the interpreter where to find the program header table in the memory image.
2e:0170│          0x7fffffffe3d0 ◂— 0x4 AT_PHENT 
2f:0178│          0x7fffffffe3d8 ◂— 0x38 /* '8' */  the size, in bytes, of one entry in the program header table to which the AT_PHDR entry points.
30:0180│          0x7fffffffe3e0 ◂— 0x5 AT_PHNUM
31:0188│          0x7fffffffe3e8 ◂— 9 /* '\t' */ the number of entries in the program header table to which the AT_PHDR entry points.
32:0190│          0x7fffffffe3f0 ◂— 0x7 AT_BASE
33:0198│          0x7fffffffe3f8 —▸ 0x7ffff7dd5000 ◂— jg     0x7ffff7dd5047 holds the base address at which the interpreter program was loaded into memory. 这里是ld.so的地址
34:01a0│          0x7fffffffe400 ◂— 0x8 AT_FLAGS
35:01a8│          0x7fffffffe408 ◂— 0x0 一些flag位，但暂时没有用？？
36:01b0│          0x7fffffffe410 ◂— 9 /* '\t' */ AT_ENTRY
37:01b8│          0x7fffffffe418 —▸ 0x400470 (_start) ◂— xor    ebp, ebp # the entry point of the application program to which the interpreter program should transfer control.这就是entry_point的地址 
38:01c0│          0x7fffffffe420 ◂— 0xb /* '\x0b' */ AT_UID 
39:01c8│          0x7fffffffe428 ◂— 0x3e8 the real user id of the process.
3a:01d0│          0x7fffffffe430 ◂— 0xc /* '\x0c' */ AT_EUID
3b:01d8│          0x7fffffffe438 ◂— 0x3e8 the effective user id of the process.
3c:01e0│          0x7fffffffe440 ◂— 0xd /* '\r' */ AT_GID
3d:01e8│          0x7fffffffe448 ◂— 0x3e8
3e:01f0│          0x7fffffffe450 ◂— 0xe AT_EGID
3f:01f8│          0x7fffffffe458 ◂— 0x3e8
40:0200│          0x7fffffffe460 ◂— 0x17 AT_SECURE 
41:0208│          0x7fffffffe468 ◂— 0x0 if the program is in secure mode (for example started with suid). Otherwise zero.
42:0210│          0x7fffffffe470 ◂— 0x19 AT_RANDOM
43:0218│          0x7fffffffe478 —▸ 0x7fffffffe4c9 ◂— 0x3f2d4c26e658aa6f # 16 securely generated
random bytes.
44:0220│          0x7fffffffe480 ◂— 0x1a AT_HWCAP2
45:0228│          0x7fffffffe488 ◂— 0x0 contains the extended hardware feature mask. Currently it is 0, but may contain additional feature bits in the future.
46:0230│          0x7fffffffe490 ◂— 0x1f AT_EXECFN
47:0238│          0x7fffffffe498 —▸ 0x7fffffffefbe ◂— 0x6667682f746e6d2f ('/mnt/hgf') a pointer to the file name of the executed program.
48:0240│          0x7fffffffe4a0 ◂— 0xf AT_PLATFORM
49:0248│          0x7fffffffe4a8 —▸ 0x7fffffffe4d9 ◂— 0x34365f363878 /* 'x86_64' */ a string containing the platform name.
```
再往下就是一些数据了。比如保存环境变量的字符串，这里我居然看到了一个环境变量是SHELL=/bin/bash，如果能泄露到这个变量，那连输入/bin/sh也不用愁了，不知道部署到服务器会怎么样。可能到docker里环境变量就全没了吧
```
pwndbg> find 0x7fffffffe260, 0x7ffffffff000-1, "sh"
0x7fffffffeda6
1 pattern found.
```
我随便打开了一个程序，初始时的栈离底部是3488字节。

## sysdeps/x86_64/start.S函数
32位：
首先清空ebp
再调用__libc_start_main，然后就是hlt这个指令，opcode是f4。。。这个难道不是让cpu停止工作的指令吗。。。
_start的函数代码在sysdeps/x86_64/start.S
这个hlt处的代码的注释写道：/* Crash if somehow `exit' does return.	 */
也就是说libc_start_main是不会返回的。因为它调用了exit
```
STATIC int
LIBC_START_MAIN (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
		 int argc, char **argv,
#ifdef LIBC_START_MAIN_AUXVEC_ARG
		 ElfW(auxv_t) *auxvec,
#endif
		 __typeof (main) init,
		 void (*fini) (void),
		 void (*rtld_fini) (void), void *stack_end)
```
参数有main函数，argc，argv，\[辅助向量数组\]，init函数 finit函数，rtld_finit函数，栈末尾指针
所以，当我们在rop中违法调用_start的时候，栈上的第一个数当成了argc，argv也错位了，剩下的参数还好
## csu/libc-start.c
__libc_start_main的主要功能：
处理关于setuid、setgid程序的安全问题
启动线程
把fini函数和rtld_fini函数作为参数传递给at_exit调用，使它们在at_exit里被调用，从而完成用户程序和加载器的调用结束之后的清理工作
调用其init参数
调用main函数，并把argc和argv参数、环境变量传递给它
调用exit函数，并将main函数的返回值传递给它

但是这里发现_start函数有一个很诡异的动作就是push rax push rsp。不知道是不是有意为之。总之栈上在argc上面就多了这两个数据。
其实是因为之前栈进行了对齐，这里push的rax是没有用的数据。而push的rsp是第七个参数stack_end指针，它要被放在栈上。rax就是为了保证对齐的。
接下来call __libc_start_main 栈上多了第一个返回地址。在argc上面一点点的__start+41这样的地址就是第一个返回地址。然而其实它并不会返回。

接下来则是dl去延迟绑定__libc_start_main。这里push了序号2和linkmap。之后进入_dl_runtime_resolve_xsavec它建立起了第一个栈。也就是说，这里保存了一个为0的rbp。所以按照栈帧回溯，最终是要回溯到0的。。。

我调试时发现源码上方写道忽略了fini参数，让fini参数在__cxa_atexit注册

## csu/elf-init.c (libc_csu_init)
libc_start_main函数的init参数被设置成了csu_init函数。csu函数先是调用了_init函数，再是循环调用init_array的函数指针，传入的参数和main函数一样
这里的init函数也在程序中
![start main call graph](../imgs/startmaincallgraph.png)

在我的ida中，它也叫init_proc。程序极其短，就算是汇编也没有几行。调用完gmon函数就完了，所以它也就是一个设置profiling的函数

```
sub     rsp, 8          ; _init
mov     rax, cs:__gmon_start___ptr
test    rax, rax
jz      short loc_592
call    rax ; __gmon_start__

loc_592:
add     rsp, 8
retn
```

> gmon_start函数。如果它是空的，我们跳过它，不调用它。否则，调用它来设置profiling。该函数调用一个例程开始profiling，并且调用at_exit去调用另一个程序运行,并且在运行结束的时候生成gmon.out。

回到csu_init，看来还是靠csu，它去调用每个init array
```
push    r15
push    r14
mov     r15, rdx
push    r13
push    r12
lea     r12, __frame_dummy_init_array_entry # init array
push    rbp
lea     rbp, __do_global_dtors_aux_fini_array_entry # finit array的开始地址就是init array 的结束地址！
push    rbx
mov     r13d, edi
mov     r14, rsi
sub     rbp, r12 把init array的结束地址减去开始地址再除以8得到数组大小
sub     rsp, 8
sar     rbp, 3
call    _init_proc # 调用init
test    rbp, rbp
jz      short loc_7B6 # 如果数组大小是0就提前跳转结束
```
也就是csu_init 就是负责调用每一个init数组的。
那init数组里到底有什么？一般是frame_dummy
为什么csu结尾有那么多的pop? 这是为什么? 难道csu是用汇编写的??
其实不是, csu的源码如下
```c
void
__libc_csu_init (int argc, char **argv, char **envp)
{
  /* For dynamically linked executables the preinit array is executed by
     the dynamic linker (before initializing any shared object).  */

#ifndef LIBC_NONSHARED
  /* For static executables, preinit happens right before init.  */
  {
    const size_t size = __preinit_array_end - __preinit_array_start;
    size_t i;
    for (i = 0; i < size; i++)
      (*__preinit_array_start [i]) (argc, argv, envp);
  }
#endif

#ifndef NO_INITFINI
  _init ();
#endif

  const size_t size = __init_array_end - __init_array_start;
  for (size_t i = 0; i < size; i++)
      (*__init_array_start [i]) (argc, argv, envp);
}
```
有pop可能只是编译器用到了太多寄存器去实现这个函数, 所以先保存在栈上吧....

在x86-64-psABI.pdf的Program Loading and Dynamic Linking里面的Dynamic Linking的最后部分：
```
5.2.2 Initialization and Termination Functions
The implementation is responsible for executing the initialization functions specified
by DT_INIT, DT_INIT_ARRAY, and DT_PREINIT_ARRAY entries in
the executable file and shared object files for a process, and the termination (or
finalization) functions specified by DT_FINI and DT_FINI_ARRAY, as specified
by the System V ABI. The user program plays no further part in executing the
initialization and termination functions specified by these dynamic tags.
```
## frame_dummy
> 接下来frame_dummy函数会被调用。其目的是调用__register_frame_info函数，但是，调用frame_dummy是为了给上述函数设置参数。这么做的目的是为了在出错时设置unwinding stack frames。
但是在我这ida里，它调用的是register_tm_clones
https://stackoverflow.com/questions/41274482/why-does-register-tm-clones-and-deregister-tm-clones-reference-an-address-past-t
原来这两个函数是gcc的函数，其实不是libc的，
https://stackoverflow.com/questions/34966097/what-functions-does-gcc-add-to-the-linux-elf

总之它们是为了提供多线程的原子类操作transaction memory的，在这里其实就是什么也不干，不会调用_ITM_registerTMCloneTable

## 返回
csu_init结束返回


## exit函数

```
Functions that were registered with the atexit or on_exit functions are called in the reverse order of their registration. This mechanism allows your application to specify its own “cleanup” actions to be performed at program termination. Typically, this is used to do things like saving program state information in a file, or unlocking locks in shared data bases.

All open streams are closed, writing out any buffered output data. See Closing Streams. In addition, temporary files opened with the tmpfile function are removed; see Temporary Files.

_exit is called, terminating the program. See Termination Internals.
```
把fini函数和rtld_fini函数作为参数传递给at_exit调用，使它们在at_exit里被调用


## 其他：栈调用规范

这个规范里面有关函数调用的部分值得好好读读，下面是之前探究system函数rop调用失败的情况。有机会再补充。

The x86-64 System V ABI guarantees 16-byte stack alignment before a call, so libc system is allowed to take advantage of that for 16-byte aligned loads/stores.

找这个标准
https://stackoverflow.com/questions/18133812/where-is-the-x86-64-system-v-abi-documented
找到了
https://github.com/hjl-tools/x86-psABI/tree/hjl/master

下载下来看第18页，里面的图显示了需要在调用函数时对齐16字节。
也就是call的时候的push rip占了8字节，然后函数开头保存ebp占用8字节，刚好16字节。
所以只要遵循了这个函数调用就可以正常使用system函数了.