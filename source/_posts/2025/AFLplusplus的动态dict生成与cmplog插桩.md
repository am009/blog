---
title: AFLplusplus的动态dict生成与cmplog插桩
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- Fuzz
- Read
---

AFL++的动态dict生成与cmplog插桩

<!-- more -->

## CmpLog模式的使用

正常的Fuzzing模式是编译插桩后的二进制，然后使用afl-fuzz程序指定路径启动fuzz。

使用cmplog模式后，除了普通的插桩二进制，还需要额外编译一份cmplog插桩的二进制。通过编译的时候指定`AFL_LLVM_CMPLOG=1`启动这种特殊的插桩。然后在fuzz的过程中，使用`-c path-to-cmplog-binary`指定第二个二进制程序。在fuzz过程中会执行该二进制，通过插桩记录下所有的比较指令，以及字符串比较。

### CmpLog模式的插桩

AFL++采用了cmplog，和动态dict在功能上有所重合。这篇文章[《Improving AFL++ CmpLog: Tackling the bottlenecks》](https://arxiv.org/pdf/2211.08357.pdf)介绍了cmplog当前实现的不足。

不像honggfuzz使用clang的标准插桩，AFL++使用了自己的自定义的Pass进行插桩。

字符串比较函数相关的插桩逻辑如下

- 根据函数签名是否像strcmp函数，即两个指针参数（isPtrRtn）。像memcmp函数，两个指针加上一个整数（isPtrRtnN）。整数需要是32或者64位大小。

```cpp
          bool isPtrRtn = FT->getNumParams() >= 2 &&
                          !FT->getReturnType()->isVoidTy() &&
                          FT->getParamType(0) == FT->getParamType(1) &&
                          FT->getParamType(0)->isPointerTy();

          bool isPtrRtnN = FT->getNumParams() >= 3 &&
                           !FT->getReturnType()->isVoidTy() &&
                           FT->getParamType(0) == FT->getParamType(1) &&
                           FT->getParamType(0)->isPointerTy() &&
                           FT->getParamType(2)->isIntegerTy();
```

- 主要区分的核心函数，isMemcmp isStrcmp isStrncmp。根据函数名判断，收集常见的函数名以及其他的类似封装`xmlStrcmp`。`isPtrRtnN`直接看作是一种Memcmp

| 插桩函数名                                 | 变量名              | 被插桩的函数组           |
| ------------------------------------- | ---------------- | ----------------- |
| __cmplog_rtn_hook                     | cmplogHookFn     | 其他未归类的签名类似的函数     |
| __cmplog_rtn_llvm_stdstring_stdstring | cmplogLlvmStdStd | llvmStdStd        |
| __cmplog_rtn_llvm_stdstring_cstring   | cmplogLlvmStdC   | llvmStdC          |
| __cmplog_rtn_gcc_stdstring_stdstring  | cmplogGccStdStd  | gccStdStd         |
| __cmplog_rtn_gcc_stdstring_cstring    | cmplogGccStdC    | gccStdC           |
| __cmplog_rtn_hook_n                   | cmplogHookFnN    | Memcmp 和未归类的同签名函数 |
| __cmplog_rtn_hook_strn                | cmplogHookFnStrN | Strncmp           |
| __cmplog_rtn_hook_str                 | cmplogHookFnStr  | Strcmp            |
|                                       |                  |                   |

相关的插桩函数的实现在`instrumentation/afl-compiler-rt.o.c`：

插桩的共享内存相关的数据结构主要在[`cmplog.h`](https://github.com/AFLplusplus/AFLplusplus/blob/c1e4b8f7f6f1f95a94c2340de4f57a998c90f094/include/cmplog.h)里。包括两个数组。用具体哪个下标是通过hash函数处理得到的，然后拿着这个下标访问headers和log两个结构体数组。

```cpp
struct cmp_map {
  struct cmp_header   headers[CMP_MAP_W];
  struct cmp_operands log[CMP_MAP_W][CMP_MAP_H];
};
```

首先会在headers里面设置一些信息。包括 hits 击中次数。shape，对比的两边的大小。type，对比的两大类类型，cmp表示一些cmp指令，比较整数，rtn表示一些memcmp或者strcmp这种内存比较。attribute 属性，比如是switch还是什么strcmp。

```cpp
struct cmp_header {  // 16 bit = 2 bytes

  unsigned hits : 6;       // up to 63 entries, we have CMP_MAP_H = 32
  unsigned shape : 5;      // 31+1 bytes max
  unsigned type : 1;       // 2: cmp, rtn
  unsigned attribute : 4;  // 16 for arithmetic comparison types

} __attribute__((packed));
```

然后在log里面存入两个被对比的值。由于256bit大小的数字大多都能支持，这里留了两个256bit数字的大小。

```cpp
struct cmp_operands {

  u64 v0;
  u64 v0_128;
  u64 v0_256_0;  // u256 is unsupported by any compiler for now, so future use
  u64 v0_256_1;
  u64 v1;
  u64 v1_128;
  u64 v1_256_0;
  u64 v1_256_1;
  u8  unused[8];  // 2 bits could be used for "is constant operand"

} __attribute__((packed));
```

对于rtn这种字符串比较的类型，则会将log数组的项目重新强制类型转换为cmpfn_operands类型。存储两个最长32字节的字符串，以及他们的长度。

```cpp
struct cmpfn_operands {

  u8 v0[32];
  u8 v1[32];
  u8 v0_len;
  u8 v1_len;
  u8 unused[6];  // 2 bits could be used for "is constant operand"

} __attribute__((packed));
```

总之，插桩代码会用上面的方式，在cmplog共享内存中存储关于程序的比较的信息，用于后续Fuzz算法使用。


### CmpLog模式的Fuzz算法

主要的入口是在`src/afl-fuzz-redqueen.c`文件的`input_to_state_stage`函数中。

1. 执行colorization阶段，更新buf
    1. 在保证执行路径完全相同（coverage map完全相同），且执行时间大致不变的情况下，尽量替换输入中的字节。
2. 在原始输入和colorized输入上，运行待测程序得到两份cmplog。(由于coverage map完全相同，可以假设cmplog完全按顺序一一对应。)
    1. cmplog数据分别存在afl->orig_cmp_map和afl->shm.cmp_map
3. 处理每个log项目
    1. 如果是cmp类型则执行cmp_fuzz函数中的逻辑
    1. 如果是rtn类型则执行rtn_fuzz函数中的逻辑
4. 如果定义了CMPLOG_COMBINE，则更新virgin_bits。

its_fuzz函数是对执行fuzz（common_fuzz_stuff）的封装，而common_fuzz_cmplog_stuff则是会执行额外指定的cmplog插桩的binary。

> CmpLog状态的第一步是着色阶段。在着色过程中，输入的每个字节都被替换为同一类型的另一个字节，一系列被替换的字节称为一个Taint区域。最初，整个输入被视为一个Taint区域，这个着色后的输入被传递给目标程序。当执行路径的结果哈希与原始执行路径的哈希不同时，Taint区域会被分成两半并分别处理，否则该区域会被保存。着色后的结果是一个着色输入，其中每个Taint区域中的字节都被替换为同一类型的另一个字节，并且执行路径的哈希等于原始输入。对于哈希的构建，使用了“trace_bits”和“map_size”。

> `rtn_extend_encoding()` 函数替换复制值的第一个字节，并使用这个修改后的输入执行目标。这将持续进行，直到达到Taint区域长度的最后一个字节，并且只要比较的另一侧的值等于将要被替换的输入字节。其背后的想法是，如果操作数值与该输入字节具有 I2S（input-to-state） 关系，则值应该相等。对于 INS 比较，这必须同时适用于原始输入和着色输入。对于 RTN 类型，这只需要适用于其中之一。请注意，只有在禁用转换时才会执行此 I2S 检查，因为当转换启用时，此检查不再适用。

这里主要关注的是什么条件下会增加dict：在cmp_fuzz或者rtn_fuzz的结尾的判断中，

- 对于cmp，严格一些，需要o（colorized之后的cmplog项目中的v0/v1）和orig_o（原始输入中的cmp项目中的v0/v1）相等，**且**没有发现新的崩溃（`!found_one`），或者是纯文本的buffer。调用的是`try_to_add_to_dict`。
- 对于rtn，宽松一些。需要o和orig_o相等，**或**没有发现新的崩溃（`!found_one`）或者是纯文本的buffer。调用的是`maybe_add_auto`。

由于没有发现新的崩溃是很常见的，所以对于rtn，基本上都会调用maybe_add_auto。