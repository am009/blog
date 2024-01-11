---
title: Fuzz快速上手指南
date: 2023/12/23 11:11:12
categories:
- Hack
tags:
- Fuzz
---

Fuzz快速上手指南

<!-- more -->

不得不说，学习使用LibAFL的过程，就是学习Fuzz架构的过程。基于LibAFL的baby fuzzer教程，就可以了解到fuzzer的架构。LibAFL是基于Rust语言，高度可自定义的组件化fuzzer。他们的开发者正尝试用LibAFL复刻AFL++，libfuzzer等多个知名fuzzer，这足以说明LibAFL的强大。

Fuzzing的很重要的一部分就是调试崩溃和修复漏洞。这个和fuzz本身一样重要。这个暂时没涉及

本文涵盖以下内容：

- AFL的coverage map设计
    - PCGUARD模式
    - cmplog/input to state/redqueen
- Mutators
  - honggfuzz中的Mutator
  - AFL中的splice和havoc
  - MOpt

### 学习资源

- [fuzz101](https://github.com/antonio-morales/Fuzzing101)
- [AFL-TRAINING](https://github.com/mykter/afl-training)。

## Fuzzer 架构

### LibAFL的主要模块

- Executor：指定InProcess（当前地址空间）还是ForkServer（执行其他程序），或者InProcessFork。
- Observer：主要功能是存储和上次执行有关的易失性数据，然后后续的Feedback可以获取这些observer，确定。
- Feedback：包括确定是否interesting的普通feedback，以及确定是否达到目标的Objective Feedback。默认Objective的不会放到queue里。
  - 因为反馈的是一个bool值，feedback可以使用逻辑运算符feedback_or和feedback_and。
- Input和TestCase。Input就是普通的buffer，或者复杂的结构体。TestCase则是Input的高级封装，方便存到磁盘。
- Corpus：一般至少两个，一个存储感兴趣的Input，一个存达到Objective的Input。
- Mutator：对输入进行变换。Mutator之间可以进行组合。
- Generator：如果没有初始输入的话，可以用它来生成随机字符串作为初始输入
- Stage：阶段，在每次fuzz时执行一次。比如mutationStage，负责mutate，并测试mutate出来的结果是否interesting。其次可以增加一些Observe的stage，对当前输入进行观察，获得数据，比如TracingStage，执行程序，然后用Observer收集一些信息。CalibrationStage，给输入定一个合适的超时时间。

然而即使是InProcessExecutor，为了保证持续fuzz，有时会用一个`SimpleRestartingEventManager`，以从而在崩溃的时候重新起当前进程。在初始化的时候调用`SimpleRestartingEventManager::launch`，它内部会把主线程挂住，然后起子线程做真正的fuzzing，然后每次在子线程死了的时候重新fork。这也导致lldb调试的时候得额外加上`,"initCommands": ["settings set target.process.follow-fork-mode child"]`。

## AFL的coverage map设计

即使是几年后的现在，最初[AFL的coverage map设计](https://github.com/google/AFL/blob/master/docs/technical_details.txt)依然被沿用。

本质上AFL的coverage map还是衡量的边覆盖率。很自然会想到，通过插桩在运行时维护一个从每条边到它执行时经过的次数的map。然而这样做了之后可能占内存很多，也很慢。和Hash函数的思想类似，通过放弃可解释性，放弃追溯具体是那条边的能力，以增加效率。

AFL在每个分支处插桩以下代码。每次fuzz执行前将对应coverage区域清零，执行结束后shared_mem（默认64 kB）被填充了每条边执行次数的信息。

```c
  cur_location = <COMPILE_TIME_RANDOM>;
  shared_mem[cur_location ^ prev_location]++; 
  prev_location = cur_location >> 1;
```

在执行时，每个分支处的随机数就代表了当前分支。每个随机数右移一位后就代表它作为prev_location时的值。异或操作得到的随机数代表具体的一条边。可以想象这些边作为下标均匀分布在数组的槽中，冲突概率较小，从而大致数组每个元素代表每个边经过的次数。

这个方法有明显的几个优点

- 插桩时只需要生成随机数，时间和内存效率高
- 运行时仅对每个分支增加几条汇编指令的开销，时间效率高。
- shared_mem内存大小可调，内存效率高。

对于fork server架构的fuzzer，这块内存可以直接作为共享内存，直接读取子进程的coverage map。对于in proc的fuzzer，则直接读取对应内存位置。此时该shared_mem区域可能作为一个全局数组/指针，在静态/动态链接时作为符号导出，fuzzer通过名字引用。

**coverage map过小可能产生大量冲突？**：确实有可能，而且这也是AFL++的[lto模式的优化](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.lto.md)之一。随机数的随机特性，肯定会没那么均匀，导致部分数组成员同时记录了多条边，然后有些数组成员没有代表任何边。

LTO指link time optimization.平时编译是按照编译单元，一个文件一个文件直接转换为汇编，然后在汇编层链接。在LTO模式下，编译器每次编译一个文件会先不生成汇编，而是保留便于分析的IR到object文件。在链接到一起的时候，即生成可执行文件的时候，此时有了程序的所有IR代码，能做的优化也更多了。

LTO允许部分优化推迟到链接后进行。之前（AFL-LLVM模式）是分文件生成汇编，此时每个编译单元内，插桩逻辑单独运行，为分支生成随机数。如果能把插桩推迟到最后一起做，则可以在生成随机数的时候相互协调，保证coverage map里每个数组成员都只代表一条边。

LTO模式不仅代码优化更好，而且插桩也有无冲突的优点，因此它也是使用AFL++时推荐能开则开的模式。

**coverage衡量指标的优化**

AFL++也提供了很多[其他模式](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.llvm.md#3-options)

- Context sensitive coverage：区分不同caller。比如函数A和B都调用了C，那么C里的同一条边，因为考虑的调用者，反馈的覆盖信息就会不同。 `map[current_location_ID ^ previous_location_ID >> 1 ^ hash_callstack_IDs] += 1`
- n-gram branch coverage：根据之前从哪个边走过来的，区分不同的当前边：`map[current_location ^ prev_location[0] >> 1 ^ prev_location[1] >> 1 ^ ... up to n-1] += 1`

**其他**

- 多线程并发修改coverage map时可能存在竞争问题：设置`AFL_LLVM_THREADSAFE_INST`

### 边执行次数的bucket设计

coverage map的成员是1字节，即会记录这条边从0到255的执行次数。AFL进一步细分，把执行次数分为如下8类。即，在程序执行完后，对每个字节应用一个函数，从0-255映射到0-7。

> 1, 2, 3, 4-7, 8-15, 16-31, 32-127, 128+

虽然信息量减少了，但是这样能关注更重要的信息：从执行1次，到执行2次或三次，有可能就有了新的突破，对一个执行了很多次的循环的次数增加则不需要那么敏感。


### AFL的Queue设计

除了每次程序执行的coverage map，AFL维护一个全局的coverage map，包含了之前执行时遇到的所有边。如果某次执行发现了新的边，则当前的输入就会认为是interesting的，放入队列中。

另外，每次执行都会设置超时时间。背后的思想是，即使有些输入能增加1%的coverage，但是会花了上百倍的执行时间，从务实的角度考虑也要排除这种情况。说不定后面就有更高效的方式覆盖到相同的代码了。毕竟速度和效率是fuzzing的精髓之一。


## AFL的forkServer设计

被fuzz的程序，首先我们一般得给他弄成输入一个buffer的形式。比如经典的`LLVMFuzzerTestOneInput`。然后main函数可以选择从第一个参数读入文件名，然后把文件内容喂给这个函数，也可以选择stdin读入直接解析。具体怎么弄是根据AFL命令行，指定被测程序的命令行那部分。就是`afl++ <afl flags> -- @@`这种东西，然后会把`@@`替换成被测程序的输入。

```
int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  ...
}
```

但是这样每次都要启动一次程序，启动成本就上去了。所以说这种fork的fuzzer就不如一些inProcess的fuzzer速度快了吗？也并不是，因为引入了forkServer的机制和[persistent mode](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md)。

**Snapshot LKM**：LibAFL的开发者发现fork太慢之后，提出了一种方法：增加一个专为fuzz设计的snapshot内核模块。代码在[AFL-Snapshot-LKM](https://github.com/AFLplusplus/AFL-Snapshot-LKM)。可以给一整块地址打快照，fuzz结束后再恢复。但是还是下面的Persistent Mode用的更多。

**Persistent Mode**：通过在编写harness的时候增加`__AFL_FUZZ_INIT();`，在fuzz的函数外面增加循环(`while (__AFL_LOOP(10000)) {`)，以及相关的重置状态的操作，这样并不是每次执行都重新fork，而是每次fork可以fuzz很多次。从而接近in process的fuzzer的效率，带来10-20x的速度提升。此外，通过共享内存（`__AFL_FUZZ_TESTCASE_BUF`）直接发送fuzz输入（比如[这里](https://github.com/AFLplusplus/LibAFL/blob/74783c2027c0ccf54a90f03ab21e7c1aae4a1b2f/libafl/src/executors/forkserver.rs#L1232)），绕过文件系统，也可以增加效率。

编写persistent mode的harness的时候，需要调用初始化`__AFL_INIT();`，fuzz循环`while (__AFL_LOOP(10000)) {`。在`__AFL_INIT();`内部调用了AFL提供的特殊函数，其中包括了父进程和子进程的握手机制。

**Pipe&Timeout**：现在往往使用的ForkServer都是开启persistent mode，同时也开启timeout功能的。所以这一块他们具体是怎么沟通交流的呢？这里有三个进程：Fuzzer，Driver Parent（下面简称DP），Driver Child（下面简称DC）。首先Fuzzer进程启动，调用了ForkServer管理端，管理端会在一个特殊的FD（`#define FORKSRV_FD 198`）创建pipe用于和DP通信。在实例化ForkServer的时候启动DP。DP直接根据环境变量映射Coverage Map共享内存，然后进行功能协商和握手，可以设置coverage map大小，是否共享内存传递输入，是否开启cmplog模式。

总结每轮Fuzzer想要运行一次的循环里，相关的pipe通信：
1. Fuzzer写入四字节`was_killed`，表示上次是否杀死了child，指示DP是否要保留上次PID。把DP从read FORKSRV_FD的阻塞中恢复。
1. DP fork出DC并写入DC的PID，把pid传给fuzzer。
  1. Timeout时：Fuzzer向DC发送kill，此时依然需要读取DP的wait返回值。
  1. 正常执行：（persistent mode）DC给自己发送SIGSTOP，暂停运行，让DP从wait中恢复。
1. DP wait结束写入四字节wait返回值
  1. DP在写入前通过`WIFSTOPPED`判断是否DC stop了自己，是的话才能在下一次运行时复用DC的PID。否则重新fork。
  1. 崩溃发生时：Fuzzer可以通过wait的返回值判断是否发生崩溃。
1. DP读四字节`was_killed`阻塞等待下一次运行

这里DP也是一个稳定的进程，不会因为fuzz导致的崩溃而退出，它通过Pipe，把PID和wait的结果都传递给fuzzer了，相当于把fork出来的子进程还是交给fuzzer管理。如果这个子进程超时了，也是由fuzzer发送sigkill信号杀进程。这里还有一个小的条件竞争，即DP同时等到了DC的stop信号，同时Fuzzer那边也发送过kill信号。这种情况下还是需要重新fork。

**握手**

1. fuzzer在forkserver模式下启动 被fuzz的进程的时候会设置一系列环境变量，包括`__AFL_SHM_ID`等。
  - `__AFL_SHM_ID`: 传递edge coverage map的共享内存ID
  - `__AFL_SHM_FUZZ_ID`: 传递输入的共享内存ID
  - `__AFL_CMPLOG_SHM_ID`: 传递cmplog map的共享内存ID
1. 被fuzz进程在`__AFL_INIT();`时，根据环境变量，设置好共享内存等变量。同时会向fuzzer发送四字节的flags，里面设置了自己支持的功能对应的比特位。


```cpp
cc_params[cc_par_cnt++] =
    "-D__AFL_LOOP(_A)="
    "({ static volatile const char *_B __attribute__((used,unused)); "
    " _B = (const char*)\"" PERSIST_SIG
    "\"; "
    "extern int __afl_connected;"
    "__attribute__((visibility(\"default\"))) "
    "int _L(unsigned int) __asm__(\"__afl_persistent_loop\"); "
    // if afl is connected, we run _A times, else once.
    "_L(__afl_connected ? _A : 1); })";
```
这里`_B`是一个字符串`#define PERSIST_SIG "##SIG_AFL_PERSISTENT##"`，不太重要。关键是`_L`的定义。这里的意思大概是转而调用了__afl_persistent_loop函数。


## AFL的其他模式

### 什么是PCGUARD？

PCGUARD是AFL的默认模式，名字[可能源自插桩pass的名字](https://stackoverflow.com/a/68919788/13798540)。它就是简单地在每个基本块前插桩。

在这个clang的[文档](https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-data-flow)里提到了相关的插桩选项。比如`-fsanitize-coverage=trace-pc-guard`就是普通的coverage插桩。

### 什么是persistent mode？

[AFL++的文档](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md)介绍了，persistent mode是指在fork server架构下，每次fork会fuzz多次target。

这里指，AFL会在运行二进制后，让程序停在main函数，然后每次fuzz前尝试复制这个进程去fuzz，以避免初始化的开销，有点类似fork。如果程序有复杂的初始化逻辑，还可以选择合适的位置，推迟这里“fork”的点。

### `cmplog`/`input to state`/`redqueen`

当fuzzing遇到多字节比较的时候，很难通过随机变换生成对应的字节出来。[AFL++的fuzzing_in_depth.md](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#b-selecting-instrumentation-options)里有一些简单的介绍，有两种方式解决：

- `laf-intel`/`COMPCOV`：将多字节比较，拆分为很多单字节比较。
- `cmplog`/`input to state`/`redqueen`：当出现这种多字节比较的时候，对应的字节会被反馈给AFL++，作为关键词用于mutate填充。这种方式比前一种更加高效。

AFL的cmplog模式，需要编译两个binary，一个设置环境变量`AFL_LLVM_CMPLOG=1`后开启了cmplog的，以及一个没有开启cmplog的binary。平时执行的时候就用没有开启cmplog的，减少开销。

LibAFL里通过`-fsanitize-coverage=trace-pc-guard,trace-cmp`开启了cmp比较语句的插桩。在[Observer的pre_exec](https://github.com/AFLplusplus/LibAFL/blob/bc91436ef47a6dfad7f19d0ab1a5af5fac97556b/libafl/src/observers/cmp.rs#L300)中执行清空map的操作。

#### CmpLog的步骤

正如文档里所说，细节得看[RedQueen paper](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04A-2_Aschermann_paper.pdf)，另外还有这篇文章[《Improving AFL++ CmpLog: Tackling the bottlenecks》](https://arxiv.org/pdf/2211.08357.pdf)介绍了cmplog当前实现的不足。

然而，根据[《Improving AFL++ CmpLog: Tackling the bottlenecks》](https://arxiv.org/pdf/2211.08357.pdf)，AFL里面的CmpLog模式，和RedQueen paper里的还不太一样。RedQueen paper里用很大篇幅介绍绕过checksum的事情，CmpLog没有实现。另外，实现的细节也不太一样。但是还是根据这篇paper里描述的解读

CMPLog最基本的思想是，程序里面很多字节和输入里的字节有对应关系。比如我在程序里发现了一个四字节比较，一边是“abcd”，一边是“\x00ELF”，同时发现这个abcd在程序输入中也出现了。此时我就可以增加一个mutate关系：把“abcd”mutate成“\x00ELF”。

**Colorization**：（见paper的第5页右侧，Ⅲ.A.v）简单应用上面的方法可能出问题：比如我们种子输入，可能里面有很长一段全都是`\x00`这种字节。如果生成一个“\x00\x00\x00\x00”到其他东西的mutate，那么会导致可以应用的地方太多。如paper中的例子所示：

![为什么需要引入Colorization阶段？](redqueen1.png)

Colorization阶段基于现有输入变换得到一个新的输入，相比原来的输入多了更多的熵。同时我们可以在两个输入上观察应用的行为，在应用mutation时，可以要求两边在相同的输入都可以这样mutate。即，在两个输入上观察那个四字节比较，比如输入1是“\x00\x00\x00\x00” 和“\x00ELF”，然后“\x00\x00\x00\x00”在input的offset是offset1，colorized的input是“abcd”和“\x00ELF”，然后“abcd”在input里的offset也是相同的offset1。

**详细算法细节：LibAFL**：这里解读LibAFL里自己实现的CmpLog技术。

CmpLogMap为什么是二维数组：首先是最简单的`CmpLogInstruction`结构体，包含了两个被对比的值（u8,u16,u32,u64），表示两个值被cmp了。但是整个Map是一个二维数组（[代码](https://github.com/AFLplusplus/LibAFL/blob/0b38fabeb009a201cce87cc0e56a57f5ea4199dc/libafl_targets/src/cmps/mod.rs#L253)），默认高32，宽65536。宽很好理解，不同插桩点的cmp值。高则代表了单个位置的cmp指令，被执行多次，可能遇到多次不同的cmp情况。

从Map到Metadata：`CmpObserver`在执行结束的时候，会处理（[代码](https://github.com/AFLplusplus/LibAFL/blob/bc91436ef47a6dfad7f19d0ab1a5af5fac97556b/libafl/src/observers/cmp.rs#L131)）Map，得到可能的mutation作为metadata存入state。而处理的时候有简单的过滤。CmpLog具体怎么处理，由于RedQueen paper它自己不是很开源，这一块具体怎么做还是非常模糊。LibAFL这里简单过滤了loop相关的cmp。在遍历单个cmp的多次执行的时候，如果发现有一边在大部分的执行时是简单的递增或者递减。那么这个cmp不是很有意义。

具体mutate的时候，则是双向mutate。随机选取输入的一个offset，向后搜索。如果当前位置的字节，取出数字来，刚好等于两边cmp entry里面任意一边的值，则把当前字节替换成另外一边的值。这意味着有可能会使得本次mutate直接skip。

CmpLogMap被定义为一个rust的全局变量（见[这里](https://github.com/AFLplusplus/LibAFL/blob/0b38fabeb009a201cce87cc0e56a57f5ea4199dc/libafl_targets/src/cmps/mod.rs#L397)），在[cmplog.c](https://github.com/AFLplusplus/LibAFL/blob/0b38fabeb009a201cce87cc0e56a57f5ea4199dc/libafl_targets/src/cmplog.c)文件里有相关的处理操作。这个cmplog.c是通过rust编译的，见[libafl_targets/build.rs](https://github.com/AFLplusplus/LibAFL/blob/bc91436ef47a6dfad7f19d0ab1a5af5fac97556b/libafl_targets/build.rs)。在构建过程中，有一些变量是动态生成到一个rust文件的：

```rust
        /// The size of the edges map
        pub const EDGES_MAP_SIZE: usize = 65536;
        /// The size of the cmps map
        pub const CMP_MAP_SIZE: usize = 65536;
        /// The width of the aflpp cmplog map
        pub const AFLPP_CMPLOG_MAP_W: usize = 65536;
        /// The height of the aflpp cmplog map
        pub const AFLPP_CMPLOG_MAP_H: usize = 32;
        /// The width of the `CmpLog` map
        pub const CMPLOG_MAP_W: usize = 65536;
        /// The height of the `CmpLog` map
        pub const CMPLOG_MAP_H: usize = 32;
        /// The size of the accounting maps
        pub const ACCOUNTING_MAP_SIZE: usize = 65536;
```

**详细算法细节：AFL++**

![Cmplog原理示意图](cmplog.png)

因此，我们现在有两个输入。每个输入执行后都有一系列的CmpLog的项。每个项里面记录了两个比较的值。但是因为不确定哪边是Magic Header，哪边是和我们input有对应的值。因此我们需要对左右分两种情况。

<img src="cmplog-algo1.png" width="60%"/>

<!-- ![cmplog-algo1](cmplog-algo1.png) -->

```
算法
for replication, pattern in CmpLogs
  for taint_region in regions
    for byte in taint_region
      考虑从当前字节到region结尾这一段
      对两种情况调用处理函数
```

正如这个函数所示

<img src="cmplog-algo2.png" width="60%"/>

<!-- ![](cmplog-algo2.png) -->

```
解释上图里的算法
尝试一个字节一个字节的来，把当前考虑的这一段替换成对应magic byte（有点子字符串匹配算法的感觉）。每替换一个字节运行一下。
for 每个字节
  检查两个对应关系：
    1 （I2S对应关系）当前改的字节，和当前cmplog的pattern对应上 （o_pattern[i]==orig_buf[idx+i]）
    2 改了这个字节之后运行，当前cmplog确实这个字节跟着改变了。（pattern[i]==buf[idx+i]）
    如果对应关系不成立则break。
  替换当前字节并运行。
```

对于常量的比较，即使是8字节比较，但是实际上比较的可能是4字节，甚至2字节单字节。再加上大小端序，对每个比较都能分出下面8种情况。

![](bswap.png)

**算法的缺点：执行爆炸**

所以，之所以会执行爆炸，主要是两点：

1. 常量比较，由于考虑了多种比较情况，尤其是2字节单字节的情况，导致出现特别简单的单字节替换，在整个输入（实际上taint bytes）里有太多的替换的地方。
1. buffer比较，由于每次只替换一个字节，导致如果有很多连续的相同字节，有可能有阶乘的复杂度。和子字符串比较算法里出现的情况类似。每次替换一下就执行一下程序，导致执行次数爆炸。

#### LibAFL实现细节

## Mutation

[LibAFL里对Mutator的介绍](https://aflplus.plus/libafl-book/core_concepts/mutator.html)很简单。Mutator是一个[Trait](https://docs.rs/libafl/0.11.1/libafl/mutators/trait.Mutator.html)，包含两个函数，mutate和post_exec。同时，[Stage](https://aflplus.plus/libafl-book/core_concepts/stage.html)也很重要，Mutational Stage也会对语料库里的输入进行修改。

### [honggfuzz](https://honggfuzz.dev/)的mutator

honggfuzz的mutator就放在最外层的`honggfuzz/mangle.c`和`mangle.h`里。在mangle.c的最底部即可看到一个mangleFuncs数组。

一个值得注意的是，里面的随机数有很多是用的选取函数`mangle_getLen`，会经常倾向于更小的数字。

1. [mangle_Shrink](https://github.com/google/honggfuzz/blob/88709ce60f45ee13666a2628f03467c57429c7db/mangle.c#L687)：随机选取一个offset，然后选取一个长度（大部分情况是0-16随机选，1/16的可能性是0到<当前offset到末尾的距离>里选），然后删掉从这个offset到offset+len的数据。
1. [mangle_Expand](https://github.com/google/honggfuzz/blob/88709ce60f45ee13666a2628f03467c57429c7db/mangle.c#L675)：同上选取一个长度：（大部分情况是0-16随机选，1/16的可能性是当前offset到<程序最大输入长度>里选）。然后把从offset开始的数据向后搬运len个距离，如果要求了printable，则会把中间新增的字节填充为空格。
1. mangle_Bit：修改某个bit
1. mangle_IncByte、mangle_DecByte：将某个字节加1或者减1。如果要求是printable就麻烦一点，在printable范围内带模加减。
1. mangle_NegByte：将某个字节取反。
1. mangle_AddSub：随机选择1，2，4字节范围的值看作整数，然后生成一个范围内的，可正可负的值，加上去。
1. mangle_MemSet：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，随机选一个byte，然后有50%概率memset，50%概率插入（此时类似Expand）。
1. mangle_MemClr：同上，但是byte选0（printable的时候选空格）
1. mangle_MemSwap：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度。操作两次得到两个offset和长度，取较小的长度，然后交换内容。
1. mangle_MemCopy：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，这样选出一块内容。随机找个offset，有50%概率覆盖，50%概率插入（此时类似Expand）。
1. mangle_Bytes：在buf中随机插入或覆盖1-2个随机字节。
1. mangle_ASCIINum：在buf中随机插入或覆盖一个十进制（可带负号）数字。
1. mangle_ASCIINumChange：在buf中找到一个现有的十进制数字，然后修改。有8种可能：加1，减1，乘2，除2，覆盖为随机值，加减0-256的随机数，取反。
1. mangle_ByteRepeat：选取一段范围，插入或者覆盖，重复随机offset里的某个字节。
1. mangle_Magic：有220个magic值，随机选里面的插入或者覆盖。
1. mangle_StaticDict：随机选dict里的值插入或者覆盖。
1. mangle_ConstFeedbackDict：需要开启cmpFeedback模式，然后从cmp feedback数组里面随机选一个值出来。
1. mangle_RandomBuf：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，这样选出一块内容。然后插入或者填充随机的字节。

### AFL中的splice和havoc

> 第一次进行变异的种子文件首先要进入deterministic stage，在这一阶段每个变异操作会对种子文件的每个byte或者bit进行变异，以此生成庞大数量的测试用例来测试目标程序。在结束了deterministic stage后，进入havoc stage，AFL从变异操作中随机选Ro个对种子文件进行变异，并使用变异后的测试用例来测试程序。第三个阶段是splicing stage，进入这一阶段的判断条件很苛刻，如果AFL变异了seed pool中的所有种子文件，得到的测试用例仍然没有发现新的interesting test cases，AFL才会执行这一阶段，splicing stage只执行一个变异操作：随机选取另一个种子文件，将它和当前文件的部分内容拼接在一起，然后重新进入havoc stage进行变异。（摘自[InForSec通讯](https://www.inforsec.org/wp/?p=3950)）

AFL的[文档](https://afl-1.readthedocs.io/en/latest/user_guide.html)里有下面的介绍：

> havoc - a sort-of-fixed-length cycle with stacked random tweaks. The operations attempted during this stage include bit flips, overwrites with random and “interesting” integers, block deletion, block duplication, plus assorted dictionary-related operations (if a dictionary is supplied in the first place).

havoc是很多操作的叠加。

> splice - a last-resort strategy that kicks in after the first full queue cycle with no new paths. It is equivalent to ‘havoc’, except that it first splices together two random inputs from the queue at some arbitrarily selected midpoint.

splice会选择两个输入，在某个位置把它们拼接起来。

其他介绍：

- [AFL-变异策略](https://www.zrzz.site/posts/49460ecb/)

## Schedule

Fuzzing的调度分为两个方面，

- Input调度：对每个queue里的testcase，mutate、执行、判断是否interesting，这三个步骤一起作为一轮。调度策略决定执行多少轮。
  - `StdMutationalStage`：获取一个从零到最大mutate次数（默认128）的随机数，mutate这些次数。（代码在[这](https://github.com/AFLplusplus/LibAFL/blob/d53503b73ea0425ffdcfbc467c167b02632077b6/libafl/src/stages/mutational.rs#L190)）
  - Power Schedule：给每个queue中的输入打一个energy分，根据分数决定在这个输入上做mutate的次数。
    - `StdPowerMutationalStage`使用的是`CorpusPowerTestcaseScore`。
    - `EcoPowerMutationalStage`使用的是`EcoTestcaseScore`
- Mutator调度：仅用单个mutator，mutate一次，可能不足以发现新的路径。在单次mutate内部，可以叠加mutator。用哪个mutator多一些，用哪个mutator少一些，是否组合，怎么组合。
  - `ScheduledMutator`：内部封装多个其他mutator，在1到迭代次数（默认7）中随机选择一个数字，每次随机选个mutator，对单个input mutate这些次数，返回上层作为一次mutate。（代码在[这](https://github.com/AFLplusplus/LibAFL/blob/d53503b73ea0425ffdcfbc467c167b02632077b6/libafl/src/mutators/scheduled.rs#L104)）

例1：对于下面的代码：

```rust
// Setup a randomic Input2State stage
let i2s = StdMutationalStage::new(StdScheduledMutator::new(tuple_list!(I2SRandReplace::new())));
```

我们可以解读为：随机叠加`I2SRandReplace`1-7次作为单次mutate。mutate并执行1-128次。

例2：对于下面的代码

```rust
// Setup a MOPT mutator
let mutator = StdMOptMutator::new(
    &mut state,
    havoc_mutations().merge(tokens_mutations()),
    7,
    5,
)?;

let power = StdPowerMutationalStage::new(mutator);
```

我们可以解读为：将havoc和`tokens_mutations`(cmplog)合并作为mutator，使用MOpt调度Mutator的叠加，每个testcase上的尝试次数使用`StdPowerMutationalStage`调度。

### (Mutator调度) MOpt: Optimized Mutation Scheduling for Fuzzers 

[InForSec通讯](https://www.inforsec.org/wp/?p=3950)有些相关的介绍。

MOpt这篇paper，改进了AFL里面havoc阶段的Scheduler的过程。

LibAFL里也实现了这篇paper的方法。

### LibAFL代码里的mutation流程

LibAFL的mutation stage的主要逻辑在这个[`perform_mutational`](https://github.com/AFLplusplus/LibAFL/blob/d53503b73ea0425ffdcfbc467c167b02632077b6/libafl/src/stages/mutational.rs#L118)函数。

- 首先确定为了当前输入，打算进行多少次mutate，存入变量num。
- MutatedTransform机制，可以根据testcase和state转换输入，目前似乎只是从TestCase解出input。
- 循环num次
  - 复制一份输入
  - 对输入调用mutator变换：mutator可以返回MutationResult::Skipped，表示变换失败，直接continue
  - 运行输入，判断是否interesting（`fuzzer.evaluate_input`）。
    - evaluate_input_with_observers：执行程序，执行observer，执行`scheduler.on_evaluation`，执行`process_execution`。
      - process_execution这里会判断是否找到bug（`ExecuteInputResult::Solution`），或者interesting（`ExecuteInputResult::Corpus`），或者都不是。
        - 对于interesting的输入，复制
  - 执行Mutator的post_exec
