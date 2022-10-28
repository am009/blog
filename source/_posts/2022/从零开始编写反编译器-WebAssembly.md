---
title: 从零开始编写反编译器-WebAssembly
date: 2022/9/18 11:11:12
categories:
- Project
tags:
- WebAssembly
- Decompile
- Compiler
---

NotDec：从零开始编写反编译器： https://github.com/am009/NotDec 

- 2022年10月5日 项目目前处于起步阶段，希望有大佬能来一起参与开发。

<!-- more -->

### 为什么要反编译？ 二进制分析和源码分析的差距到底在哪？

**程序分析和程序变换相结合**：我一直在思考，二进制分析和源码分析的差距到底在哪？我们知道，源代码一般是不能被直接执行的，我们之所以能够在源代码这个层次上编程，得益于最初编译技术的发展。程序在编译后变成了机器能理解的代码，但是同时也变得更复杂了。从这个角度看，编译器也可以看作是一种混淆，而我们去除混淆后不仅能更容易理解，也使得静态分析更加容易。

我们知道，机器执行的是汇编指令，有各种寄存器，内存，然后高层的数据类型，比如结构体和数组，在汇编指令的层次也不复存在。但最关键的是，每个函数的局部变量，变成了线性内存中维护的栈空间。（callee saved register和寄存器也是，但是这里暂时不提）举一个很简单的例子。

```c
int arr[3] = 0;
int a = 0;
int b = 1;
int c = 2;
arr[input()] = 1;
```

如果直接分析汇编指令，就有点类似下面的形式。可以看到对数组的访问可以影响后面的所有变量。这种出现问题的情况还有很多。现在有不少针对二进制代码的定制化静态分析，但是因为处理了很多这样的复杂情况，效率上也是远远比不上源码的分析效率的。

```c
int arr[6];
arr[input()] = 1;
```

既然编译器会增加这么多复杂性，很自然的想法是怎么将编译器做的变换，逆着变换回去，变成原来的更简单的形式。这自然而然地让人想到了反编译。反编译技术的架构一般如下：除了最后的控制流分析，前面的步骤基本上就是我们想要的。

反编译器架构：

1. 前端：将字节码转为LLVM IR
2. 中端：优化与分析
   1. 分析函数参数、分析callee saved register (wasm可以跳过这个阶段)
   2. SSA构建：使得前端可以有些冗余的alloca，由SSA构建来将相关alloca消除。 （编译原理相关）
   3. GVNGCM：Global Value Numbering and Global Code Motion 优化算法，有强大的优化能力，有助于反混淆等。（编译原理相关）
   4. 内存分析：将各种通过内存访问的变量显式地恢复出来。可能要用到指针分析算法，类型恢复等。关键词：Memory SSA。
3. 后端：高层控制流恢复，将字节码转为AST，打印为高级语言的形式。

有的人可能会说反编译器可能不能保证静态分析的安全性（soundness），但是随着反编译技术的发展，很多分析手段都是较为通用的，同时能够在遇到异常情况的时候报出警告，告知相关不安全的判断。

### 当我们说转IR的时候，我们在说什么

在反编译过程中，随着分析的深入，我们的IR也从low level变得更加high level。引用《Static Single Assignment for Decompilation》第6页（1.1 source code）附近说的，源代码可以说分为几个层次：

1. 高质量，带注释源码（Well written, documented source code, written by the best human programmers.）
1. 丢失了注释，函数名和各种变量名，结构体成员名字（Source code with few to no comments, mainly generic variable and function names, but otherwise no strange constructs. This is probably the best source code that a fully automatic decompiler could generate.）
1. 同上，但是偶尔有奇怪的（底层的）东西（occasional strange constructs）
1. 没有从内存中识别变量，所有的内存访问都是通过地址计算（Source code with no translation of the data section. Accesses to variables and data structures are via complex expressions such as *(v3*4+0x8048C00). Most registers in the original program are represented by generic local variables such as v3.）
1. 同上，而且所有原始的寄存器也存在（but all original registers are visible.）
1. 同上，相当于套了一个虚拟机，整个程序就是一个巨大的switch结构，根据不同的opcode去执行不同的指令。（but even the program counter is visible, and the program is in essence a large switch statement with one arm for the various original instructions）

### 现有的wasm反编译器

正如上文所说，现有的“反编译器”，很多的问题都在于没有去理解更深入的语义。

wasm有层次化的几个语言特性，也就意味着优化程度较高的代码对更底层的使用往往更少，导致简单的折叠出来的代码也有不差的效果。但是实际上并没有多少“反编译”的工作在里面。

1. 栈 对应着SSA
1. local 对应着没有转为SSA的局部变量
1. 内存中另外维护的栈 对应着需要取地址的变量，最为复杂。

观察现有的“反编译器”：

1. wasm2c 这是wabt的一个工具，可以配套一些外围代码让wasm转换后的C语言能够运行起来。它比套虚拟机好一些，因为wasm指令还是比较简单的，可以转换为C语言指令，而不用弄一个巨大的switch case。
1. wasm-decompile 这也是wabt的一个工具，
1. wasmdec github上的一个简单的开源项目。很多这种“反编译器”一看代码量非常小，必然是不可能有完善的反编译分析的。问题也在于没有对内存有足够的分析。
1. jeb-pro 这个软件包含一个商业的反编译器，之前在安卓和二进制那边都有一些名气。二进制那边出名的反编译器大多都不怎么支持wasm和EVM，但是它似乎对EVM字节码和wasm都有支持。这种从二进制那边过来的反编译器想必基础更加扎实，效果应该也更好。

## wasm的反编译

1. 前端：wasm字节码转LLVM IR
    - wasm独特的，可静态类型检查的栈结构，需要稍微特殊地处理一下。
    - wasm独特的控制流，处理起来也不是那么简单。
2. 中端：优化与分析
   1. 分析函数参数、分析callee saved register (wasm可以跳过这个阶段)
        - wasm作为新时代的字节码标准，作为在浏览器运行中的标准，层次就比汇编高了很多，反编译也更加容易。这里主要说的是，汇编中需要指令和数据的区分，函数的识别，函数参数与保存的寄存器的处理，而这些在wasm中完全不需要。
   2. SSA构建：使得前端可以有些冗余的alloca，由SSA构建来将相关alloca消除。 （编译原理相关）（可以直接用LLVM的）
   3. GVNGCM：Global Value Numbering and Global Code Motion 优化算法，有强大的优化能力，有助于反混淆等。（编译原理相关）（可以直接用LLVM的）
      - 还有很多LLVM的pass可以用过来，可以参考RetDec用的。
   4. 内存分析：将各种通过内存访问的变量显式地恢复出来。可能要用到指针分析算法，类型恢复等。关键词：Memory SSA。
3. 后端：高层控制流恢复，将字节码转为AST，打印为高级语言的形式。

### wasm的栈处理与wasm混淆

![wasm设计时就考虑了解码为ssa](wasm-design-rationale.png)

其实底层看，wasm的栈机制和结构化控制流，和一个东西很像。在SSA形式的IR中，用于替换Phi指令的语义等价的另一种表示形式是basic block argument。[MLIR](https://mlir.llvm.org/docs/Rationale/Rationale/#block-arguments-vs-phi-nodes)也提到了，[这里](https://news.ycombinator.com/item?id=22432344)也有相关的讨论。。所以其实wasm字节码的栈部分其实应该可以直接转成SSA形式。

如果你还是不确定Phi指令和Basic Argument是一个东西的话：因为Phi指令必须在基本块开头，而且必须对每个precessor都有一个对应的值，即操作数数量和precessor数量相同。（如果你说未初始化的变量。。那可以在entry块给每个变量赋值为undef，这样必然有值存在）然后想象把Phi指令的操作数直接保存到这条边上，更进一步放到跳转过来的每个jump语句处，然后phi指令对应的value还是放在当前基本块开头，这样其实就变成了BasicBlockArgument。

那wasm的栈到底要怎么转为SSA形式呢？我们可以参考basic block argument思想，在每个block或者loop块，为每个block的argument创建Phi指令，最后再移除trivial的Phi指令即可。

wasm和其SSA的对应形式其实也是对应的。比如，在字节码层面直接进行控制流平坦化，可能遇到栈上的东西不平衡的形式。而栈对应着SSA的值，其实在OLLVM那边的控制流平坦化也会遇到处理Phi指令的问题。他们的策略是demote ssa的值，降级到普通load-store的形式，这个就对应到wasm的local了。意味着我们也应该将导致栈不平衡的值作为local处理。这三个层次从上到下，限制越来越少，即下面的层次完全可以替代上面的层次，但是同时也越来越难以分析，因为其实限制越多越利于分析。

1. 栈 对应着SSA
1. local 对应着没有转为SSA的局部变量
1. 内存中另外维护的栈 对应着需要取地址的变量，最为复杂。

**wasm的混淆自身有没有独特（specific challenge）之处呢？** 我觉得还是有的，核心其实就在转换这部分。通过先转为现有SSA，再转回Wasm的方式，最大程度复用了混淆中共通的逻辑，那么额外需要的逻辑自然就是wasm特有的挑战了。如果不转为IR直接混淆确实会将部分wasm特有的挑战和混淆逻辑混合在一起，应该是让事情更复杂了。

### wasm的控制流处理

最近出来了wasm 2.0。看了下好多复杂的东西。不过wasm 1.0 （MVP）还是非常简单的。对于每个控制流相关的指令

1. block，loop分别对应在结尾，开头，增加一个label。
1. if对应一些label和br_if，br代表直接跳转，br_if同理，根据语义找到对应的跳转目标，生成条件跳转即可。
1. br_table看似比较麻烦，看了下和LLVM的switch语句对应得非常好啊。也是根据不同的值跳转到不同的边。

### SSA IR的选择 - 为什么用这个IR？

搜了一下，wasm相关的IR有bineryen，和Cranelift。他们内部都有SSA相关的表示。cranelift作为SSA IR，和LLVM IR在很多方面是非常相似的。但是最后各种选择可能

1. bineryen：非常有名，命令行工具是wasm-opt好像。它使用的IR似乎已经有点[架构上的问题](https://github.com/WebAssembly/binaryen/issues/663)
1. Cranelift: wasmtime的编译器架构中使用的IR。也支持SSA。但是没想到它居然直接添加了类似vmContext这种隐藏的参数，同时将全局变量lower到了用offset表示。

**为什么不用现有的开源代码里面的转换部分？** 确实存在相关的项目：

1. [WAVM](https://github.com/WAVM/WAVM)和[aWsm](https://github.com/gwsystems/aWsm) 这两个都有编译器，而且也是LLVM IR的。所以里面很多转换相关的逻辑都是可以抄的。
1. WAMR wasm-micro-runtime 基于LLVM的，但是是C语言，使用LLVM-C-API，我们打算用的是C++的API。所以不太合适。

但是他们似乎也有类似的问题：

1. 转换出来的IR带有很多“无关”函数。因为这些VM的编译器要么是JIT要么是AOT，都得带上运行需要的外围函数。和我们的目标还是有所偏离的。

另外一个非常奇怪的事情是，(在各种C/C++的wasm runtime项目中，wavm和wamr)，wasm二进制格式解析也没有特别通用统一的库。大家好像都自己写了一套。（除了rust的项目，rust好像都用的wasmparser）


TODO: 内存分析，控制流恢复，AST生成