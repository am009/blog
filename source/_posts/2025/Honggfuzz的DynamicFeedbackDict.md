---
title: Honggfuzz的DynamicFeedbackDict
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- Fuzz
- Read
---

Honggfuzz的DynamicFeedbackDict机制

<!-- more -->

## 实际场景的优势

Honggfuzz实现了一套相比现有cmplog更简单的机制，却达到了很好的效果，甚至在proj4_proj_crs_to_crs_fuzzer这个benchmark上超越了其他的fuzzer。

![honggfuzz 禁用feedback dict后，效果大打折扣](honggfuzz-decdict.png)

可以看到图中，禁用feedback的hongfuzz-decdict，覆盖率降低了接近一半。正常的honggfuzz-orig，比AFL++效果还要好。

## Clang的trace-cmp选项

[Clang可以增加`--trace-cmp`选项](https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-data-flow)，在代码中出现比较指令的时候，会调用提供的插桩函数。比如下面就是8字节比较会插入的插桩函数：

```c
void __sanitizer_cov_trace_cmp8(uint64_t Arg1, uint64_t Arg2);
```

honggfuzz提供的插桩函数在`/sn640/fuzzerlog/honggfuzz/libhfuzz/instrument.c`。

### Honggfuzz的插桩逻辑

Hongfuzz的feedback dict主要分为两部分，

- 插桩部分负责将重要的内容存入dict数组，即生成dict关键词。
- 后续mutation中随机抽取数组中的词找个位置随机插入或者覆盖（即和普通的dict使用一致（虽然使用概率很高，大于50%）。

因此主要介绍插桩部分。

首先是存储生成的dict的数组。结构如下所示：

```c
typedef struct {
    uint32_t cnt;
    struct {
        uint8_t  val[32];
        uint32_t len;
    } valArr[1024 * 16];
} cmpfeedback_t;
```

这个结构在整个fuzzing过程中共享一个实例。其中第一个字段的cnt会随着fuzz过程不断增长。每个dict的值不能超过32字节大小，总数量不超过1024\*16个。后面mutation的时候就是随机抽取valArr里的成员。所以我们重点关注valArr的使用。

以`__sanitizer_cov_trace_cmp8`为例，主要有以下几个插桩逻辑：

- 每4095次才进入内部，即按概率随机抽取进入内部逻辑。
- `__builtin_bswap64`主要负责处理大小端的问题。
- `util_64bitValInBinary`很关键，仅过滤出binary中存在的内容，加入dict。因为比较运算的时候，经常是输入中的一个什么东西和程序里的常量比较。如果不加过滤，那么就是把fuzz自己得出的东西当做dict使用，意义不大。
    - 在最开始的时候，honggfuzz会整理程序中的值存到values32InBinary数组中。这个数组在`collectValuesInBinary_cb`中被初始化。它使用libdl，遍历程序（非动态链接库）中readonly的段，就直接按32字节，64字节收集起来了。然后排序，方便后续二分查找。

```c
void __sanitizer_cov_trace_cmp8(uint64_t Arg1, uint64_t Arg2) {
    /* Add 8byte values to the const_dictionary if they exist within the binary */
    if (globalCmpFeedback && instrumentLimitEvery(4095)) {
        if (Arg1 > 0xffffff) {
            uint64_t bswp = __builtin_bswap64(Arg1);
            if (util_64bitValInBinary(Arg1) || util_64bitValInBinary(bswp)) {
                instrumentAddConstMemInternal(&Arg1, sizeof(Arg1));
                instrumentAddConstMemInternal(&bswp, sizeof(bswp));
            }
        }
        if (Arg2 > 0xffffff) {
            uint64_t bswp = __builtin_bswap64(Arg2);
            if (util_64bitValInBinary(Arg2) || util_64bitValInBinary(bswp)) {
                instrumentAddConstMemInternal(&Arg2, sizeof(Arg2));
                instrumentAddConstMemInternal(&bswp, sizeof(bswp));
            }
        }
    }

    hfuzz_trace_cmp8_internal((uintptr_t)__builtin_return_address(0), Arg1, Arg2);
}
```

上面插桩获取的都是大小为4或者8的值，此外，字符串比较也非常重要。这是通过`libhfuzz/memorycmp.c`里的插桩实现的。

```
__sanitizer_weak_hook_strcmp
__sanitizer_weak_hook_strcasecmp
__sanitizer_weak_hook_stricmp
```

```c
void instrumentAddConstStr(const char* s) {
    if (!globalCmpFeedback) {
        return;
    }
    if (!instrumentLimitEvery(127)) {
        return;
    }
    /*
     * if (len <= 1)
     */
    if (s[0] == '\0' || s[1] == '\0') {
        return;
    }
    // 关键的检查，地址要是程序内部的地址。
    if (util_getProgAddr(s) == LHFC_ADDR_NOTFOUND) {
        return;
    }
    instrumentAddConstMemInternal(s, strlen(s));
}
```

对于字符串比较，也存在可能把自己fuzz随机生成的东西抓过来的问题。这里通过判断**值的地址是不是映射到二进制内部**进行筛选。一般程序输入的值都在栈上或者堆上，然后和程序自带的字符串（位于不可变的内存区域）进行比较。如下面的代码所示，如果值太长就切断到32字节。同时也会遍历已有的map找是否已经这个值已经被插入了。

```c
static inline void instrumentAddConstMemInternal(const void* mem, size_t len) {
    if (len <= 1) {
        return;
    }
    if (len > sizeof(globalCmpFeedback->valArr[0].val)) {
        len = sizeof(globalCmpFeedback->valArr[0].val);
    }
    // 检查dict数组长度不能超过收集的最大数量
    uint32_t curroff = ATOMIC_GET(globalCmpFeedback->cnt);
    if (curroff >= ARRAYSIZE(globalCmpFeedback->valArr)) {
        return;
    }

    // 如果和现有的项目完全相同就跳过
    for (uint32_t i = 0; i < curroff; i++) {
        if ((len == ATOMIC_GET(globalCmpFeedback->valArr[i].len)) &&
            hf_memcmp(globalCmpFeedback->valArr[i].val, mem, len) == 0) {
            return;
        }
    }

    uint32_t newoff = ATOMIC_POST_INC(globalCmpFeedback->cnt);
    if (newoff >= ARRAYSIZE(globalCmpFeedback->valArr)) {
        ATOMIC_SET(globalCmpFeedback->cnt, ARRAYSIZE(globalCmpFeedback->valArr));
        return;
    }

    memcpy(globalCmpFeedback->valArr[newoff].val, mem, len);
    ATOMIC_SET(globalCmpFeedback->valArr[newoff].len, len);
    wmb();
}
```

### 移植到AFL++上

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
