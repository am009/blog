---
title: NativeSummary：Java/C安卓应用无源码污点分析。
date: 2022/5/1 11:11:11
categories:
- Read
tags:
- StaticAnalysis
---

让原有的安卓污点分析工具（如FlowDroid），支持通过JNI接口调用的汇编代码（C语言代码）的分析。

不使用符号执行等动态分析技术，而是使用二进制静态分析技术。

<!-- more -->

<!-- 这是我的本科毕设。最初选题其实我非常不喜欢这个题目的，中间觉得选都选了，还是继续搞吧，但是非常难受。觉得如果想不清楚，最后写起代码来就是软件工程上的灾难。很难做到可维护性和容易理解。现在我后悔了。我要么就别做，要么就不要说自己正在做的事情的坏话，因为这只会让自己更加折磨。不要否认任何方向和课题，因为这会让你的路变窄，丧失灵活性。不做还好，一旦你必须要做这个方向，就会遇到无尽的走神，自我否认。从此以后，即使一个方向我觉得大概率有坑，稍微远离就好了，我也不会去否认它，而是保持自我怀疑，毕竟没有做就不知道是不是真的有坑。真的不能有极端的情绪，坏处很多。（说多了都是泪） -->

这是论文的“手稿”。因为用来翻译，所以一些英式中文是故意的。

这个题目可以分为两个部分
1. 逆向部分：将APK中的二进制库拿出来，分析基于JNI接口的汇编代码在干什么。
2. 跨语言部分：C语言和Java语言的差异摆在那里，如何污点分析。

通过探索得到了下面的方案：

1. 首先分析Java部分找到native函数。然后对应到二进制库中的导出函数，即找到二进制的入口点。
1. 依次分析每个JNI函数。至少把JNI API的调用提取出来。
1. 基于观察，部分JNI函数还是非常接近Java里面的一些操作的。比如`SetObjectField`对应Java里面设置对象的成员，`CallObjectMethod`对应Java里面调用函数。基于这个思想，把相关的调用转换成Java语句。
1. 为了兼容现有的框架，将生成的Java语句重打包到应用里。因为现有的（只支持Java虚拟机部分代码）的APK污点分析工具，除了都输入一个APK，也没什么额外的拓展接口了。所以通过重打包实现兼容。

（这一段还是不放进去）
Java侧分析和native侧分析很不一样。Java侧字节码含有的信息较多，大量工具直接基于JVM字节码直接分析。而native侧分析和C/C++源码层分析还是非常割裂。二进制分析往往采用符号执行、fuzzing等方式，而源码层则可以采用基于抽象解释的分析技术。我们相信，在足够的逆向工程努力下，我们也可以对二进制代码使用基于抽象解释的指针分析等技术。

## Contributions

1. 提出了一种兼容现有Java 侧分析框架的二阶段跨语言分析方法。
1. 通过使用静态分析技术分析汇编语言代码，相比于符号执行技术，提升了分析的鲁棒性（和代码覆盖率）。
1. 把现有的工具打包为了docker镜像，极大地提升了现有工具的可用性和用户友好性。
1. 


## 背景 

### Java侧污点分析框架

现有的Java污点分析框架很成熟，甚至出现了大量论文评估现有的Java污点分析框架，如ReproDroid，《Analyzing Android Taint Analysis Tools: FlowDroid, Amandroid, and DroidSafe》。这些工具利用现有的java分析技术，利用soot框架转换为Jimple这样的中间表示，分析dex字节码。然而，开发者可以通过JNI接口调用通常用C语言编写的本地代码，在java侧表现为带有native标识符但没有函数体，如Lising 1中所示。

由于二进制分析和跨语言分析的难度，现有的工具只能采取保守的黑盒策略。他们忽略了本地代码调用的副作用，并假设数据流只会在参数和返回值之间发生。 该模型仅涵盖本机函数仅进行计算但实际上它们可以调用 Java 端函数并修改全局或实例状态的情况。

### JNI的使用

当一个开发者使用JNI时，首先在Java侧声明native函数，使用javap【TODO】生成对应的C语言函数签名，编写函数体。编译为动态链接库，运行时则加载，JVM获取导入函数，增加函数映射。然而还有另外一种方式，即在JNI_OnLoad函数中使用RegisterNatives注册函数。

JNI静态绑定和动态绑定。

### 二进制分析与抽象解释

现有的往往使用符号执行。但是符号执行往往会遇到路径爆炸，我们使用抽象解释的分析能增加探索到的路径。

（分享之前选择二进制分析的相关背景知识？）
VSA是一种相比符号执行更轻量级的静态分析方法。他基于抽象解释，抽象每个程序点每个变量可能的取值。它结合了指针分析和数值分析，对程序的数值分析促进了地址访问的解析，而对地址访问的精确处理使得程序的数值分析更精确，两者相互促进。

BinAbsInspector，TODO

## 例子

> 例子要体现：
> 1. 我们能够支持对Java侧的调用。
> 1. 我们能够支持对native侧的数据流分析。Jucify做不到。
> 1. 我们能够支持高级特性。

> 例子的特点：
> 1. 我二进制分析，直接只分析了数据流部分，即使内部分为了多个函数，也能考虑到数据流，然后转换为单个函数。比如global id里面的全局数据流
> 2. 可以包含一些我们支持的高级用法。比如file leak里面的native特有的函数。

（开场怎么说？Native code分析很难？Native code对数据流分析很重要？不，为了说明跨语言分析的难点）
~~为了说明本地代码对数据流分析的重要性~~为了说明我们数据流分析的整体流程，我们给出了一个例子。恶意应用开发者能使用和实例一样的代码泄露用户隐私。和传统

（说一下App的数据流泄露流程）

（说一下我们的处理方式。）

Lising 1-2展示了一个例子。正如很多现有论文【cite】所说，本地代码已经被广泛应用。现实中的恶意应用开发者很可能使用本地代码以躲避现有分析工具的分析。

**处理JNI相关的API** 直觉是，很多JNI相关的操作都能被转换为对应的Java操作，比如，这个例子中的CallObjectMethod可以被转化为一个java method call，比如GetObjectField能被转换为Field read。

**处理native code中的数据流** 虽然在例子中不明显，但很容易想到C/C++中有更多独有的传播数据流的语言特性。很难直接找出一个简单的方式去incorporate Java和C/C++的数据流区别，特别是当出现了很深的函数调用，复杂的指针传递数据，而这些很难在java侧找到等价物。
实际上，这些对保留敏感的穿过native code的污点数据流并不是必要的。因此，我们选择不在java侧反映复杂的控制流和函数结构，仅为每个native 方法创建一个Java函数体。二进制中的数据流仅被用于解析Java操作之间的数据流。为了避免类型系统的冲突，有时需要使用自定义的类型转换方法或强制类型转换。~~，利用强制类型转换避免类型系统的约束~~。

**ID as global variable** 相关ID的缓存，也是一个难点。这意味着我们需要考虑从JNI_OnLoad到后续函数的，经过全局变量的数据流。这对我们的二进制分析提出了挑战。

**Native specific sources and sinks**现实生活的app中可能会大量的使用各种各样的使用纯Native的source和sink点，无法一次性考虑到。我们采取了动态的方式，根据二进制侧动态链接库的导入函数去动态创建为特殊的Java侧native函数声明，同时将他们导出为Flowdroid的格式，便于进一步的处理。

正如Listing3中所示，我们为open, write 和 close构建了对应的native方法，并将write标记为Sink点，从而让现有工具成功检测到隐私泄露问题。

## 相关挑战

### 二进制分析

java那边已经有很多成熟的分析工具，而二进制代码侧虽然已经有大量和成熟的研究，但一方面二进制代码的分析如今依然无法做到全自动，需要有经验的逆向工程师，另外一方面，相关的技术和源码级分析也大相径庭。二进制分析方面的自动分析，主要有符号执行，fuzzing这种把代码近似看作黑盒的工具。【扩写一下。】而

为了能够对二进制代码做静态分析，首先要借助类似Ghidra这样的二进制分析平台。【Ghidra介绍】其次是选用静态分析。我们首先考虑使用基于VSA的静态分析，选择了基于Angr的静态分析，但是后面发现它的VSA分析非常难用，疏于维护，需要自己的大量修改，且效果较差，这一点在其他paper中也有提到【cite】。在本项目进行到一半的时候，科恩实验室开源了BinAbsInspctor，于是我们重新基于BAI实现。

BAI使用了基于抽象解释的二进制分析技术，它将每个内存区域或者寄存器抽象为K-Set，相关的lattice如图所示【】。当可能的值小于等于K个时，使用一个集合抽象表示。当集合元素超过K个时，放弃精度使用Top表示。我们没有使用了BAI的Z3，因此假设所有的路径都可能，并在路径交汇的地方进行合并。此外，跨函数分析方面是k-callsite-sensitive。我们为了效率，把跨函数值设为了最低的1。没有任何widening技巧加速迭代过程，求解器直接迭代到不动点为止。

> 直接将这种基于抽象解释的方法应用于二进制分析会遇到一个关于callee saved reg和frame pointer的问题。我们早期使用BAI时遇到了这个问题，症状表现为复杂函数调用有寄存器和内存都变为了Top。

抽象解释的这种框架在落地到二进制方面会遇到非常大的困难。我们直接使用BAI的时候出现了大量的精度下降问题，即大量寄存器和栈空间都变成了TOP，分析时间也居高不下，结果调查发现了是静态分析没有很好地处理callee saved reg的问题。一主要的困难就是难以将activation record直接抽象出来。尤其是参数和callee saved reg
【加一个calling convention的图，一个函数调用的二进制指令，一个调用栈】。

callee saved register 是在调用约定中，约定在调用前后需要保持不变的寄存器，被调用者如果使用了这个寄存器，则需要在函数开头保存到栈上，函数结尾返回前恢复。另外值得注意的是，函数调用的参数，是调用者push到栈上的，被调用者也不会为参数额外分配空间，而是直接访问栈底。导致在栈上出现了这种交错的局面。即，本应属于被调用者的函数参数被放到了调用者栈上，本应属于调用者的寄存器被放在了被调用者那边。可以想象，如果抽象解释也用这种栈布局将会带来问题，在源码层可以直接分析得到的结果在二进制层将需要额外的context sensitive level.

> 在抽象解释的视角下，就好像是把部分和调用无关的局部变量保存到callee的栈上，相关的变量可能由于敏感度不足而混合。在n-callsite-sensitive的场景下，这些受到影响的变量会等效于损失一层callsite-sensitive。此外，仔细观察epilogue可以发现，最后寄存器的恢复（最后的pop指令）依赖于frame pointer的精确保留，而frame pointer本身也是callee saved register。

> 起初，callee saved寄存器由于精度不足导致frame pointer混合了其他的值。在函数结尾的时候无法正确恢复自身保存callee saved register，又会导致caller的frame pointer被损坏，最终形成了链式反应，影响整个调用链上的函数的分析。不仅影响了分析结果，还会使得消耗更多的时间才能迭代到达不动点。

想象这样的场景，分析使用了1-callsite-sensitive，frame pointer在调用其他函数的时候作为callee saved 寄存器而与来自其他函数的frame pointer混合，导致不精确，在函数结尾的时候，由于有太多种load的可能，BAI直接返回了Top。从而caller保存的寄存器也无法恢复，进一步损害caller的fp，从而形成链式反应，导致调用链上的callee saved register都出现问题，变成了TOP，直接导致很多局部变量都丢失，在函数调用后大量寄存器和变量都只剩下Top。不仅精度下降了，迭代到Top的时间也更长。

simply one level of precision degradation seems not a big deal, but when it combines with frame pointer and callee saved register, it causes a big crash on precision.

但是实际给我们带来分析困难的情况更为复杂。在函数开头的时候会有这样的情况：
即使用了framepointer，并且在函数结尾的时候基于frame pointer的值恢复到sp。问题在于，BAI里只对SP做了特殊处理，即使sp被弄坏了，每次调用前还是能恢复。但是fp就不一样了，基本上就是一个简单的指针数据

仔细观察结尾可以发现，callee saved reg是在修改sp后恢复的。寄存器的值的正确恢复依赖于fp（r4）的值的精确保留。然而fp也是普通的callee saved register，当调用其他函数的时候会被保存到栈上，由于上面提到的精度下降问题，fp的值会在调用返回的时候无法精确解析，


```
        000199cc f0  b5           push       { r4, r5, r6, r7, lr }
        000199ce 03  af           add        r7,sp,#0xc
        ...
        00019c7e fc  1f           subs       r4,r7,#0x7
        00019c80 05  3c           subs       r4,#0x5
        00019c82 a5  46           mov        sp,r4
        00019c84 f0  bd           pop        { r4, r5, r6, r7, pc }
```

1. 我们只有1-callsite sensitive，当调用链过长的时候，还是会出现merge的情况
```
A -> B -> C
D -> B -> C
```
两个C的栈上数据会被合并。栈上保存了寄存器，因此来自不同B的栈上变量会混在一起，返回到B后如果遇到内存读写，出现了不精确的问题，就会读出TOP。比如上面的frame pointer出现了问题，导致pop的值出现了问题。

我们采取的方法是，

此外，tail call也对我们的分析产生了影响。Tail call在汇编层表现为函数末尾的普通跳转，不会识别为函数调用，仿佛函数体发生了重叠。除了部分plt段外部函数调用需要特殊处理外，似乎没有很大的影响。最后值得注意的是对32位代码和64位代码的支持，因为真实应用中依然存在大量的32bit代码。虽然没有特别多的问题，但在编程的时候需要额外注意。

### 跨语言分析

1. 创建静态分析的JNI环境  -- 放到后面去



2. summaryIR的设计。



3. JNI调用的转换。

### Java侧框架解耦

为了实现对Java侧框架解耦，我们的初始想法是，能否重打包。

## 设计

在本节，我们在3.1节中介绍了NativeSummary的架构，并在之后的小节中介绍了各个模块的实现细节。

图1展示了NativeSummary的总体架构，由3个模块组成。

（TODO 画个图，表示从哪些JNI API转换为哪些Java调用）


### 映射解析模块

映射解析模块用于静态绑定的简单解析并将.so文件解压出来以便于进一步分析。在APK中，二进制代码作为动态链接库被按架构放在lib/文件夹下。首先我们会选择我们支持的架构中，对应文件夹下动态链接库数量最多的架构作为主要分析的架构，并收集.so文件中以Java开头的导出符号。其次是提取并解析Dex文件，收集Java侧代码中所有带有native modifier的函数。最后，我们会根据JNI的name mangling规范，将Java侧JNI函数对应到native侧函数入口，生成native函数的签名，并输出为json格式。如果发现有JNI_OnLoad函数也会放到json中，以便二进制分析模块处理动态解析的情况。

<!-- 新加 -->


由于项目早期是基于Angr的VSA，因此这一部分代码使用Python语言编写，但是已经移除了angr的依赖，而是使用更轻量级和高效的pyelftools模块[TODO cite]。Dex的解析使用了androguard模块[TODO cite]，由于它会做一些交叉引用分析，这一块的耗时最多。

### 二进制分析模块

**Invocation**：二进制分析模块的范围是单个so文件。把.so文件，和静态解析的json结果作为输入。首先会为当前APK创建一个Ghidra  project，并使用GhidraHeadless命令行工具导入每个.so文件，并将我们的插件提供的入口脚本作为PostScript参数。导入时Ghidra会有一些预分析，在代码段划分函数边界并创建函数，识别常见的libc导入函数，创建交叉引用等。


**JNI环境布置**：内存布局如图[TODO]所示。JNI调用需要JNIEnv\*作为参数，JNI_OnLoad需要JavaVM\*作为参数。我们的抽象解释在分析过程中可以计算出对这种结构体中函数指针的调用，因此我们只需要在地址空间中布置好相关结构体即可。首先为每个JNIAPI创建对应的外部函数，然后按照结构体的定义布置好指向刚才创建的函数的起始地址作为函数指针。启动分析时，直接将对应的参数设置为布置的函数指针的地址即可。


**不透明数据流追踪**：JNI API调用时数据流有多种情况。

- 不透明的JNI对象：根据JNI标准，这种对象通常是jobject、jclass、jmethod等指针大小的不透明类型，由GetObjectClass等类型返回，只能原样传递回相关的API。单独实现为特殊的region似乎更好，但为了实现的简洁性，我们复用了Global空间（用于整数和指针）的一块高位范围。表现为一个非常大的负数或者内核的地址空间。不透明类型本身也不该被程序操作，因此不会有太大问题。
- buffer类型：native code可能会调用malloc等内存分配函数。调用GetStringUTFChars这种api时也会返回一块可操作的内存。因此我们使用BAI的堆模型，分配一块新的Heap Region，同时（带上污点？）。但是这种方式在面对动态生成的字符串，字符串处理函数的时候还是会遇到问题。
- 整数类型：我们使用BAI的污点追踪功能，返回一个带有污点的Top值。不仅让条件判断都判断为可能满足【去掉前面说的assume all paths are possible】，同时保证了能够追踪数据流。

~~然而，还是有很多conercase无法解析。比如动态生成的数据，各种字符串处理函数，strcat。【TODO】~~

**动态注册解析**：JNI_OnLoad函数的动态注册往往是简单地按照规范使用RegisterNative函数，传入作为全局变量的含有动态注册信息的结构体。因此也出现过基于静态扫描寻找结构体的动态注册解析方式。我们对JNI_OnLoad函数运行相同的静态分析，在RegisterNative函数建模时获取动态注册的结构体。

**缓存的ID**【找个更专业的词】：使用JNI接口时有一种缓存MethodID，classID的编程模式可以提高性能【cite】，但也给我们带来了新的挑战。我们将JNI_OnLoad相关的调用也放入summaryIR中，保存JNI_OnLoad对全局变量的修改。当其他函数使用到这种全局变量的值的时候，在解析时会直接引用到JNI_OnLoad中的指令。表现在IR中：允许其他函数引用JNI_OnLoad中API调用的返回值。在后续的类型分析中从而正确解析相关的类型。


**总结** 
1. 启动分析前，我们还根据注册信息为每个入口函数设置对应的signature信息，并
1. 如果有JNI_OnLoad函数，则先分析它，并在对RegisterNatives函数的建模中记录动态注册信息。最后为新增的函数设置参数和返回值类型。
1. 根据参数类型设置好对应的抽象值，启动静态分析。分析过程中，外部函数建模代码负责记录外部函数调用，并处理返回值。
1. 分析完成或超时后，我们获取调用点处的寄存器和内存情况，解析调用参数，解析返回值，创建summary IR并输出。

### summary IR

<!-- 新增 -->
我们模仿编译器的设计，设计了一套简单的IR用于表示二进制分析的结果，语法如图所示【参考 c-summary写下】。体现外部函数调用，和数据流在函数参数，调用参数，调用返回值，函数返回值之间的数据流关系。【或者直接说，和相关的数据流关系】。主要的直觉是，表示当前注册的JNI函数可能调用到的所有外部函数（包括JNI接口），以及参数，返回值和外部函数调用的参数之间的数据流关系。分析单个JNI函数时可能native侧会调用大量的函数，而我们最后在IR中只转换为一个函数。IR中没有控制流相关的结果，因为我们没有找到很好的方式提取数据流。

指令的顺序是按照在静态分析时第一次遇到的顺序。所以可能会出现，前面的指令引用到后面指令的返回值的情况，给后续Java语句生成带来了一些挑战。此外，为了处理【缓存的ID】的情况， 我们还允许引用JNI_OnLoad函数里得到的值。

一个函数的summary IR的语法可以被以下的语言表示
```
<Module>        :=      <Function>*
<Function>      :=      'define' <Type> '@' <FunctionName> '(' <Param>* ')' '{' <Instruction>* '}'
<Instruction>   :=      <CallInst> | <PhiInst> | <RetInst>
<CallInst>      :=      <Value> '=' 'call' *ID* <Value> ( ',' <Value> )*
<RetInst>       :=      'ret' <Value>
<PhiInst>       :=      <Value> '=' 'phi' <Value> ( ',' <Value> )*
<Value>         :=      <Param> | 'null' | 'top' | NUMBER | STRING | '%' ID | '@' ID
<Param>         :=      <Type> ID
<Type>          :=      ID

'void' | 'int' | 'long' | 'short' | 'byte' | 'char' | 'float' | 'double' | 'boolean' | 'null' | 'array' <Type> | 'object'
```

### Java语句生成模块


由于二进制和java语言之间的差异，我们无法找到直接转换的方式，而是基于观察发现，部分JNI接口的使用模式对应着一些Java语句的操作，做尽力而为的转换。相关的转换见表【】

但是在具体的转换过程中，我们发现还是有很多代码逻辑需要在生成函数体之前完成。因此我们模仿编译器中端架构，将代码组织为了Pass的形式

java语句生成模块的相关Pass如图所示

```
预加载分析 -> 外部函数调用转换 -> JNI类型分析 -> 函数体生成
```

**JNI类型分析**：在JNI调用中有这样的编程模式，获取classID，methodID后传入相关的API。我们将ID相关的调用映射到对应的Soot对象，便于后续分析的使用。然而在现实生活中的例子里也会出现相关的字符串没有在二进制分析处被正确解析的情况，会导致本分析无法解析出对应的ID，影响后续的相关Java操作的生成。

此外，IR还需要考虑缓存ID的编程模式。

函数体生成：调用Soot的API，创建Jimple代码，利用

**未知导入函数的调用**：二进制代码可能会因为引入了相关库函数而导入了其他动态链接库的函数。因此我们难以完整地考虑所有外部函数。我们创建了一个java类【叫什么来着】，将未知的导入函数对应创建一个static函数，从而将相关的调用转换为Java函数的调用。

自动类型推断：创建java侧函数需要提供函数签名。但是C语言导入函数只有名字。我们需要推断出它们的签名。对于已知的常用库函数，比如libc，Ghidra会自动设置签名，我们通过Ghidra的函数签名接口获取函数签名。但是对于未知的库函数怎么获取参数数量？

TODO，搞完几个native_leak再回来写。


**APK重打包**：Soot能直接加载APK，并输出为APK格式。它首先排除在排除列表中的类，将所有加载的dex字节码转换为jimple ir，再将加载的字节码重新转换为字节码。然而，我们在加载部分真实应用的库函数代码时遇到了一些报错（eg: android.support.constraint.solver.LinearEquation.replace）。我们试图让soot保留部分代码，但是需要对soot代码结构有较大修改。我们最后的方案是首先让soot输出dex文件，然后根据需要修改的class，将dex文件和原有APK中的dex重打包，从而最大程度减少对原有代码的修改，同时增加效率。

## Evaluation 实验和思路

RQ1 是测试集，比较完善。增加表对比支持的JNI函数的功能性。

1. 背景：真实应用数据集上，使用的sources和sinks。
1. 成功跑出的数量，增加的native边数量，增加的native相关的数据流。发现的易用性问题等各种问题。
1. 我们跑出的flow是否准确？找一下一些我们native相关的flow，稍微分类一下。


## Evaluation

> 各个模块的代码量说一下，列个表？【TODO】
我们

各个模块的代码量如图【TODO】所示，我们通过回答三个RQ来进行评估。

- RQ1: NativeSummary在测试集中表现如何
<!-- - RQ2: NativeSummary在真实应用数据集上的效率如何？ -->
- RQ2: NativeSummary在真实应用数据集中是否有助于污点分析。

### RQ0 native code in the wild

按照本地代码中外部，java侧函数调用的频率，展示一个相关的统计。说明和数据流相关的函数调用有很多，就算没有也可以辅助逆向。

### RQ1 Benchmark

这一节中，我们evaluate NativeSummary by comparing with Jucify and JN-SAF on two benchmarks: 1. NativeFlowBench from JN-SAF[] 2. NativeFlowBenchExtended designed by us.

**NativeFlowBench**：NativeFlowBench是[JN-SAF]中提出的，人工构造的App benchmark，用来测试JN-SAF的性能，并且被后续工作【μDep】沿用。我们从中排除了3个使用了NativeActivity的app，因为支持起来麻烦而且在现实生活中使用的很少。值得提及的是，我们包含了ICC相关的App，因为我们的方法能够直接支持它。

NativeFlowBench只考虑到了最为简单的几种native code的使用方式，而且关注的是对Java侧的操作，基本上每个测试用例都可以直接翻译为Java函数，无法反映真实世界中native code的使用。我们省略了其中三个App，即NativeActivity的App，因为我。
~~Jucify也提出了自己的benchmark，但没有较好的分类，且包含了字符串混淆这种favor符号执行的用例。~~

Jucify[TODO cite] also brings up its own benchmark. We didn't include it because it lacks good classification, and includes test cases that favor symbolic execution like string obfuscation.

我们在NativeFlowBench上的结果如图所示【】。

| Benchmark | Result | Benchmark | Result |
| :--- | :---: | :--- | :---: |
| native_complexdata | ◯ | icc_nativetojava | ◯ |
| native_compexdata_stringop | xx | native_heap_modify | ◯ |
| native_dynamic_register_multiple | ◯ | native_leak | ◯ |
| native_leak_dynamic_register | ◯ | native_leak_array | ◯ |
| native_method_overloading | ◯ | native_noleak | ◯ |
| native_multiple_interaction | ◯ | native_noleak_array | ◯ |
| native_multiple_libraries | ◯ | native_nosource | ◯ |
| native_set_field_from_arg | ◯ | native_source | ◯ |
| native_set_field_from_arg_field | ◯ | native_source_clean | ◯ |
| native_set_field_from_native | ◯ |  |  |

Jucify没有跑NativeFlowBench，因为NativeFlowBench中大量使用了`__android_log_print`这个native侧leak函数，而Jucify没有考虑native leak的情况。他们没有考虑native的sink点，即android_log_print函数

NativeSummary能处理绝大多数的例子，达到和JN-SAF基本相同的准确率。native_source_clean这个例子中，imei被保存为一个数据对象的成员，这个对象被传入native method中对应成员被重新赋值为常量字符串，最后这个成员变量被打印。我们使用Jadx手动查看了重打包的APK中的代码，正确反映了对应语义，因此我们认为这个误报是FlowDroid的精度不足所致，而JN-SAF的Java侧分析框架是Amandroid[TODO]，因此没有这个问题。~~为了验证，我们重新从源码编写了一个对应Java代码的应用，FlowDroid依然能够检测出来。~~

NativeFlowBench is far from being representative of real world usage of JNI interface. So it can barely reflect native code usage in real-world apps.  By manually examing real world codes, we summarized other cases.

**NativeFlowBenchExtended**：基于我们对real world apps中native code使用的观察，我们发现了很多NativeFlowBench没有考虑到的更为复杂的JNI API的用法，并总结为了一个Benchmark：

each testing one perspective of our newly discovered inter-language challenges

- native copy：将传入的Java字符串转换为C语言字符串，并使用memcpy复制，再创建一个新的Java字符串返回。这个用例能够测试对Native自身的数据流的追踪。
- native encode：基于上一个用例，但将复制改为base64编码。提高了对native侧数据流追踪的要求。

- native file leak, native socket leak：使用NDK提供的API写入文件，进行检测。
- native global id：在JNI_OnLoad中预先计算JNI调用中使用的到的Method IDs，Class IDs，以boost性能。它要求分析器能够处理使用全局变量的跨函数数据流。这个用例来自对【】应用的用法的观察。这个case来自我们对【】应用的源码的观察。
- native handle：基于在【那个开源gif库】中观察到的使用模式，将一个malloc得到的堆指针作为jlong类型返回到java侧作为句柄，传入其他API使用。



把那边毕设的表格移过来

在NativeFlowBench上我们能达到和JN-SAF一样的准确率。在NativeFlowBenchExtended上，我们准确率更高。

Benchmark app只能覆盖到JNI接口的一部分使用方法，而现实应用中，本地代码的使用要复杂得多，因此我们没有计算相关的准确率。

由于不知道这些复杂情况在真实应用中的分布，因此在这些测试数据集上的准确率难以反映真实应用中的分析情况。更不用说，真实应用中还可能有没考虑到的情况，以及交互使用导致的更复杂的情况。因此我们不计算在这些数据集上的准确率。

**RQ1 Answer**: 我们的工具在NativeFlowBench上达到了和效果最好的工具一样的precision。在NativeFlowBenchExtended中，我们能handle很多其他工具没有考虑到的情况。


### RQ2

**数据集**：

~~测试效率的数据集，我们使用了JN-SAF采用的NativeFlowBench，以及我们自己设计的拓展数据集。对于效率和真实数据集上的表现，~~
~~我们排除了不包含native方法，或不包含so的apk。~~

我们在两个真实应用的数据集中测试我们的工具，并和JN-SAF,Jucify对比，如表【展示数据集和初筛结果】所示，分别是F-Droid数据集【cite，并且标上访问日期】，和Malradar数据集【cite】。
我们首先运行了FlowDroid作为baseline。然后，我们运行每个工具，将其发现的敏感信息流与baseline做对比。我们没有和μDep对比，因为它不仅需要运行一个安卓模拟器，较为heavy weight，同时它还有部分脚本是基于IDA，这个商业软

我们将每个工具打包了并发布了为docker image，使得任何人都可以非常便利地通过一行命令内启动各个工具。

<!-- μDep通过使用基于fuzzing的方法，提取native函数的参数和返回值之间的数据流关系。 -->


**数据集预处理** 我们通过同步F-Droid repo的方式下载F-Droid数据集。repo中包含相同App的不同版本，我们进一步做了去重处理。为了筛选出我们感兴趣的，包含有native code的app，我们筛选出至少包含一个shared object(.so file)同时至少有一个方法带有native modifier的app。
此外，我们发现在F-Droid数据集中有大量flutter应用，尽管这种应用的APK中包含native库，但这种应用的代码在javascript字节码文件中，我们的分析无法产生有用的结果，因此我们将其过滤掉，通过匹配libflutter.so文件名。

<!-- Native Activity的情况？ -->

**实验的设置**：
我们的实验跑在在服务器（两个64核Epyc 7713，256G内存）上，但是我们通过使用docker的"--cpus 1"flag限制了CPU为单线程的性能，同时通过docker的"--memory=32G"flag限制了最大可用内存为32G。

**Sources和Sinks的选取**：数据流结果会大量受到Sources和Sinks文件的影响【要不要cite】。我们基于TaintBench中从各个现有工具中合并得到的sources和sinks点，然后，我们将它转换为每个工具能接受的的格式，并对它们做出相应的修改以让它们使用。看似这个source和sink过于verbose，但正如后文所示，真正的穿过native的flow数量并不多。

<!-- 它删去了过于trivial的点（String和常见数据结构的方法）。然后，我们将Sources和Sinks文件转换为每个工具的格式，并对它们做出相应的修改以让它们使用。 -->

<!-- source和sink点对分析的效果有很大影响。【引用论文】直接使用flowdroid默认的ssource，sink效果并不好。我们首先基于taint bench合并的source和sink点，然后通过手动查看分析结果，删除了大量trivial的结果涉及的source和sink点。 -->

首先，我们对每个工具统计了能够成功地完整输出污点数据流结果的app数量，如表【todo】所示，因为在面临真实世界中的app时，很多应用在分析时产生了报错。我们把成功输出taint flow结果视为一次成功的运行。其中有271个app能同时被所有tool成功分析。然后，我们统计了每个工具输出的数据流总数量。

更进一步地，我们分析了每个工具的二进制分析部分的输出，统计了他们分析的的被注册的native函数的数量，以及在分析中发现的对java的调用边数量。
- 可能有人会问，java调用边能代表分析的效果吗，但在JNI函数的使用中，对Java侧的调用是最主要的，


此外，我们尝试了直接比较增加native分析前后，报告出来的污点数据流的变化情况。更具体地，比较jucify，nativeSummary和Flowdroid。但我们发现，即使在完全相同的设置下，flowdroid也会表现出约17%的数据流变化，远大于真正因为native分析而导致的数据流变化，因此我们放弃了这种对比方式。

JNSAF发现的flow似乎远小于其他基于FlowDroid的工具，这一点在miuDep和【analyzing three】中也被证实了。一方面Amandroid的检查结果会在完全相同的输入下有很大波动。另一方面，由于是按需的分析，binary分析会在java侧分析遇到native函数时启动，因此也会影响bianry分析的函数数量。这也可能是为什么JN-SAF的总分析时间也相对较短。

此外，我们发现，当Jucify的binary分析超时时，相关的数据处理会被直接跳过，导致现有的分析结果也无法被Java侧分析利用。因此其实只有很少部分【】的apk真正被修改了。绝大多数的app都相当于直接运行flowdroid。

（再画个表，包含flow的数量，和native有关的flow数量，ns里额外包含和Native相关的flow数量）

#### 用户友好性

现有工具似乎并没有很好地为真实世界中的应用做准备。
Jucify没有输出任何flows的具体信息，而仅仅打印了是否有任何flow through native。我们通过修改了源码让它把污点流输出xml格式。此外，它使用了自定义的Sources和 Sinks格式，而不是使用flowdroid的格式，且Sources和sinks文件位于jar包内部，更改它需要重新编译。

JN-SAF自从2018年12月后就不再更新，并基于一个使用python2的旧版本angr。当我们正在使用真实世界中的应用测试JN-SAF的时候，发现大多数的App(91.32% on F-Droid dataset, 88.51% on Malradar dataset) 报错说“loadBinary can not finish within 1 minutes”【todo找一下】。我们通过人工查看JN-SAF的源码，发现JN-SAF在反编译APK的dex字节码时，硬编码了一个1分钟的超时时间，这一点绝对不用户友好。因此我们修改了它和其他两处超时时间，并重新编译工具。（无法轻易port到最新版angr）
此外，可能是使用了旧版的apktool，依然有【TODO】数量的APK在解包APK时出错。


#### Dataflow results


**JN-SAF**：

JN-SAF检测到的flow总数量过少，经过调查，我们认为原因是JN-SAF使用的Java侧分析框架Amandroid的不稳定性在分析real world apps时，正如[《Analyzing Android Taint Analysis Tools: FlowDroid, Amandroid, and DroidSafe》](https://people.ece.ubc.ca/mjulia/publications/Analyzing_Android_Taint_Analysis_Tools_TSE_2021.pdf)里所confirm的，Amandroid 在同一环境中每次独立运行时报告的数据流数量变化很大。

<!-- 工具的介绍移到introduction里？ -->
**Jucify** Jucify仅分析了native到java的调用边缘们，但它没分析控制流，或参数和返回值之间的数据流。它通过猜测参数和返回值之间的数据流的去生成bodies，而且实现也不够完善，这意味着它生成的数据流几乎不能被信任。这可能也是为什么它在实验中相比baseline差异最大。（它的实现也不完善）

Jucify不用户友好。

~~我们其中一个初始动机就是现有的工具无法在真实应用数据集中得到有用的结果，因为太多的应用都超时了。~~


<!-- 初步的映射解析数据分析？多个架构的选择情况？ -->

<!-- We summarized the number of shared objects in different ISAs in the apps of
S2 and S3. There are 15,203 native libraries in dataset S2 and
S3. 73.0% of them (11,096 shared objects) are in ARM/ARM-
64 (armeabi, armeabi-v7a, and arm64-v8a), and 21.1% of them
(3,215 shared objects) are in X86/X86-64. Only around 5.9%
are in MIPS/MIPS-64, which we do not support analyzing. On
the other hand, in dataset S2 and S3, we find no native Activity
component, which we fail to resolve and is also reported to
be very rare in the datasets of JN-SAF -->

<!-- 我们和JN-SAF和Jucify做对比。我们统计了在F-Droid数据集上和Malradar数据集上各个工具的运行时间散点图，如图所示。【Jucify自身的超时时间是二进制分析的还是总的？说一下】。有xx%的应用都超时了。对于JN-SAF【有超时时间吗，没有就我们自己设置一个。说一下】我们有 -->

![](flows.png)

### RQ3

为了进一步确认RQ2中flow的情况，我们手动分析了。

## Limitation

无法完整反映二进制侧的语义，比如控制流。

无法分析各种复杂的C++面向对象代码，此外，二进制代码还可能由Rust，go这种语言编写。

## 相关工作

（要不要画个表，但是只有这三四个工作，分别标注Java侧分析框架，分析技术（符号执行、fuzzing），加上C-Summary可以再标个是否需要源码。还可以带上C-Summary的那个反编译的工作。是否支持动态注册？）

JN-SAF，Jucify，muDep是三个最相关的工作。JN-SAF提出了一种基于summary的自底向上分析，按需分析native函数，但是也因此和Java侧框架Amandroid耦合过深。Jucify提出了基于调用的统一表示，通过angr获取对Java侧的调用，并基于猜测的方法生成函数体。因此他们仅能完善调用边，无法分析更深入的数据流。此外，他们还不支持动态注册。muDep采取了一种基于fuzzing的方法，通过不断改变参数，观察其他参数和返回值的变化，从而判断数据流关系。他们使用基于IDA scipts的自动处理脚本去解析调用边。

C-Summary 源码层。但转换后的代码仍然直接包含JNI interface的调用，需要修改Java侧框架处理这些函数调用。此外，它会保留部分JNI原语，需要修改Java侧分析框架处理。（此外，它也没那么综合完善，依然有很多复杂情况没有考虑到。）一个更进一步的工作基于现有的反编译器去分析没有源码的JNI程序。它没有开源。（我们认为这种方案在真实应用上还会遇到更多困难。）

有一些其他的没那么相关的：
1. 在早些的时候有一些C语言到Java语言的转换技术。然而，他们的目标是为了运行，而不是静态分析，因此我们认为它会生成一些复杂的用法，这样的用法现有的静态分析器会难以处理。
1. JNILight 形式化地定义了JNI接口下，同时包含C语言和Java语言的语义。【TODO从论文找更精确的定义】基于它或许能重新定义一套更完善的分析框架。



{% pdf /2022.assets/毕设论文-2022年5月17日-王纪开.pdf %}
