---
title: 如何写paper
date: 2024/01/27 11:11:12
categories:
- Notes
tags:
- Scientific
---

学术论文写作总结

<!-- more -->

### 相关工具

- [git-latexdiff-web](https://github.com/am009/git-latexdiff-web)：上传两个overleaf压缩包，运行git-latexdiff，返回一个可以看出有哪些修改的pdf。可以在其他人帮改paper后得到反馈。
- [翻译机器人Poe](https://poe.com/WritingChinese)：直接输入中文，内置提示词会让LLM将内容直接翻译为英文。

## 学习路线

没有不能克服的困难。写paper看似很难，实际上是由一些没那么难的事情组合而成。想起我之前写作卡出半天写不出一句话，有时候是逻辑理不清楚的问题，有时候是英语语法语料不足，想表达一句话不能很好地表达出来。每次写都是花费大量时间，憋不出两三句话，每次都很难受，想到就抑郁。

先写中文是有帮助的。能够清楚地知道自己往往遇到的是逻辑问题，而不是英语表达能力的问题。速度就是质量，真正好的文章是改出来的，如果不能以很快的速度进行这个过程，那么就很可能不能在自己可接受的时间范围内写出“还看得过去”的paper内容，自己也会非常折磨。可以在文档中自己分两栏，比如用个两列的列表，左边放中文，右边放英文。每次同时修改两边内容。

在分离了语言这个困难后，其次是讲求逻辑，有明确的逻辑关系。优先看自己是不是完全写清楚了，其次是考虑篇幅等其他问题。往往会发现越写越多，然后超出内容，然后开始精简，这也是写paper的必经之路。后面再删，删的时候优先选择必要的逻辑，次要的注释掉。

最后，需要注意，语言是一种艺术。必须要让自己的语言能够以最简单的方式被看懂。首先，任何逻辑上复杂的弯都最好理顺，①进一步增加自己paper的受众，让更多人能看懂，②很有可能reviewer在看你paper的时候很累，思维也并不是很清晰。其次，为了段落结构的清晰，可以通过调整句子，让自己每个段落想表达什么都明确出来。比如，介绍自己paper主要思想时，我们往往并不会写：“The main idea of our paper is that xxxx.”，但是我们却可以刻意改成这样，从而极度明确地让人知道这个段落的主要内容。从另外一方面想，把段落组织起来，利用开头句作为索引，也是方便知识查找和索引的一种方式。语言也是一种艺术，你很难知道这些句子在别人脑子里能否和得到你形成一样的想法。就像艺术要追求美一样，我们要追求逻辑上的清晰，简洁易懂。

另外，克服自己害怕的事情有一些通用的“技巧”。早上刚起床的时候做你最害怕，最不想做的事情。

### 学习方法

首先是英文写作。这两天正好在准备托福，有一道写作题，每次做完放到grammarly里检查语法错误。第一遍的时候很多语法错，有点沮丧。但是连续做了几遍这同一题之后，渐渐知道自己经常在哪些地方犯错，能够在检查的时候基本上错误没多少了。之后对英文写作就有信心多了，这一点对写paper也有帮助。

## 总体思想

- 写作要逻辑，讲求逻辑，文字是次要的。遇到自己写不出来的情况，首先问自己是不是逻辑问题，把自己的逻辑理清楚。
- Reader Oriented：
- Reproducible：重视实验结果的可复现性

写Abstract、Introduction可以留到最后。Abstract需要考虑让没有背景的人也能看懂。Introduction也不需要写得太细节，即使能很好地概况你的内容。写清楚是正文approach的事情。

如何写得更清晰：
1. 从读者熟悉的概念入手。如果要指代同一个东西，尽量用读者熟悉的那个词指代。
2. Abstract，Introduction以及Motivating Examples中，有可能写得太细节，导致读者看不懂。让其他人看看有哪些关键词太过细节，不需要那么详细说。
3. 宏观是逻辑是一条线下来，微观上段落都是总分的结构。

## 写paper

### 摘要

摘要并不是为了准确地概况paper的方法。方法不用写太细节。包括the scope, purpose, main idea, results。摘要需要易懂，让更多人能看懂，此外，让看的人知道是不是他们想要找的paper。

## Latex

**ACM模板移除Copyright**

这篇[文章](https://shantoroy.com/latex/acm-remove-copyright-information-from-first-page/)介绍了。其中[pagestyle](https://fr.overleaf.com/learn/latex/Headers_and_footers#LaTeX_page_styles)是和header和footer有关的设置，我加上好像没什么变化。

```latex
% 加在use的package后面
\setcopyright{none}
\settopmatter{printacmref=false} % Removes citation information below abstract
\renewcommand\footnotetextcopyrightpermission[1]{} % removes footnote with conference information in first column
% \pagestyle{plain}% removes running headers
```

**hyperref与链接颜色：**

不加上`hyperref`包之前，各种链接都无法点击，包括`\url{https://...}`和表格和图的`Fig.~\ref{xxx}`。直接引入`hyperref`后会使得链接外面有个浅蓝色边框。使用`\usepackage[hidelinks]{hyperref}`可以消除边框。

```
\usepackage{hyperref}
\AtEndPreamble{
	\usepackage{hyperref}
	\hypersetup{
		colorlinks = true,
		linkcolor = brown,
		anchorcolor = brown,
		citecolor = brown,
		filecolor = brown,
		urlcolor = brown
	}
}
```

使用autoref可以免去写`Figure. 1`前面的`Figure.`部分。

**IEEE 表Caption是大写**

[IEEE模板确实要求表的标题是Small Caps格式](https://tex.stackexchange.com/questions/166814/table-caption-in-uppercase-i-dont-know-why)的。但是，当你[引用了`\usepackage{subcaption}`之后会变成正常格式](https://tex.stackexchange.com/questions/387133/my-table-is-not-conforming-to-the-ieeetran-caption-table-format)。如果要提交到期刊，这是一个容易犯的错误。但是确实有很多paper的table的格式因为这个而不对。有一个[方法](https://tex.stackexchange.com/questions/154435/ieee-template-and-caption-false-option-for-subcaption-package)（[这里](https://liuzhiguang.wordpress.com/2018/01/05/get-ieeetran-to-work-with-the-subcaption-package/)也提到）可以绕过这个问题而正常使用subcaption包。

### 图表

基础样式的表格：基于 https://www.tablesgenerator.com/ 生成，选择Booktabs table style，仅选择三个横线（表顶部底部和表头）和内部竖线。

- 图表说明是否需要句号？
- 引用图表时使用波浪线：`Fig.~\ref{fig:xxx}`。波浪线代表不换行空格。
- 引用Section的时候用`\S`替代，例如`\S \ref{sssec:xxx}`
- 标号可以用`\pin`

**节约空间**

调paper空间要注意Latex的块结构。如果有个块是一个整体，比如一些RQ的Answer块，正好卡在了页面边缘，那么除非你能删改出整个块的空间，不然空间布局是不会变的。

- 可以让列表没有左边距`\begin{itemize} [leftmargin=*]`
- 如果有一些段落最后就一两个单词，可以看看怎么缩上去，节约空间。
- 如果某段结尾只有一个单词，可以段落结尾加一个`\looseness=-1`，从而增加单词之间的间距填充满这个多出来的整行。

```
\addtolength{\floatsep}{-2mm}
\addtolength{\dblfloatsep}{-4mm}
\addtolength{\textfloatsep}{-4mm}  % single column figure margin
\addtolength{\dbltextfloatsep}{-3mm}  % double column figure margin
\addtolength{\abovecaptionskip}{-2mm}
\addtolength{\belowcaptionskip}{-2mm}
```

**表太宽或太窄**

不推荐使用resize，用的话也是用`\resizebox{\columnwidth}{!}{....}`（[出处](https://tex.stackexchange.com/a/387166/308917)）

### 引用

引用格式：

> as shown by Brown [4], [5]; as mentioned earlier [2], [4]–[7], [9]; Smith [4] and Brown and Jones [5]; Wood et 
al. [7]
> 
> NOTE: Use et al. when three or more names are given for a reference cited in the text.
> 
> or as nouns:
> 
> as demonstrated in [3]; according to [4] and [6]–[9].

(摘自[IEEE Reference Guide](https://ieeeauthorcenter.ieee.org/wp-content/uploads/IEEE-Reference-Guide.pdf))

但最好不要将引用数字作为句子的成分，而是改为提作者名字，或代词指代。

### 缩写

- 对于容量，可以直接用全大写的MB，虽然MiB更精确，但是实际上都行。


## 其他

2024年1月24日 
- 表格的`\extracolsep`填满整个页，会导致边框有gap，rotate之后不对齐到中间。
- 引用网页时，使用`@misc{}`项目，如果写`url = {{xxx}}`则在后面不会显示链接。写`howpublished  = {\url{xxx}}`才会显示。[这里](https://tex.stackexchange.com/questions/171441/url-not-showing-in-references)有个相关的问题。可能`@online`和`url`搭配，而`@misc`和`howpublished`搭配使用。

2023年12月9日 ~~今天花了一个小时，就改了一个小节（半页）。有时会没有~~注意小节开头的句子和之前的连贯性。

- research做不可数名词。Some research shows that xxx
