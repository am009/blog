---
title: 如何写paper
date: 2023/12/01 11:11:12
categories:
- Notes
tags:
- Scientific
---

学术论文写作总结

<!-- more -->

## 学习路线

我自己也还在摸索，可能有不对的地方。

一定要把写作的困难分开，写作卡出半天写不出一句话，有可能是逻辑理不清楚的问题，也有可能是英语语法语料不足的问题。

其次是讲求逻辑。其实最难的就是理清楚逻辑。不过这也是必经之路，优先看是不是完全写清楚了。然后会发现越写越多。应该是必经之路，后面在删的时候优先选择必要的逻辑，次要的注释掉。

### 学习方法

首先是英文写作。这两天正好在准备托福，有一道写作题，每次做完放到grammarly里检查语法错误。第一遍的时候很多语法错，有点沮丧。但是连续做了几遍这同一题之后，渐渐知道自己经常在哪些地方犯错，能够在检查的时候基本上错误没多少了。之后对英文写作就有信心多了，这一点对写paper也有帮助。

这里不得不推荐一下[git-latexdiff-web](https://github.com/am009/git-latexdiff-web)项目，自己改paper的时候往往遇到不自信的问题，可以改完后生成diff找学长看自己改的怎么样，得到反馈。

## 总体思想

写作要逻辑，讲求逻辑，文字是次要的。遇到自己写不出来的情况，首先问自己是不是逻辑问题，把自己的逻辑理清楚。

写Abstract、Introduction可以留到最后。Abstract需要考虑让没有背景的人也能看懂。Introduction也不需要写得太细节，即使能很好地概况你的内容。写清楚是正文approach的事情。

如何写得更清晰：
1. 从读者熟悉的概念入手。如果要指代同一个东西，尽量用读者熟悉的那个词指代。
1. Abstract，Introduction以及Motivating Examples中，有可能写得太细节，导致读者看不懂。让其他人看看有哪些关键词太过细节，不需要那么详细说。
1. 宏观是逻辑是一条线下来，微观上段落都是总分的结构。

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

链接增加颜色：

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

### 图表

- 图表说明文章是否需要句号？
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

### 缩写

- 对于容量，可以直接用全大写的MB，虽然MiB更精确，但是实际上都行。


## 其他

2023年12月9日 今天花了一个小时，就改了一个小节（半页）。有时会没有注意到小节开头的句子和之前的连贯性。

- research做不可数名词。Some research shows that xxx
