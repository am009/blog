---
title: 堆复习
date: 2020/6/1 11:11:11
categories:
- Hack
tags:
- CTF
---

# 堆复习

64位时, 默认开启的fastbin范围(chunk总大小)是0x20 - 0x80
32位TODO

tcache是64个单向链表，最多7个节点(chunk)，chunk的大小在32bit上是8到512（8byte递增）；在64bits上是16到1024（16bytes递增）。
fastbin只有10个链表, 范围肯定很小, 而和smallbins有62个, 大小基本重合.

当某一个tcache链表满了7个，再有对应的chunk（不属于fastbin的）被free，就直接进入了unsortedbin中。
tcache_perthread_struct结构，一般是在heapbase+0x10（0x8）的位置。对应tcache的数目是char类型。

<!-- more -->

## 堆块结构

堆块大小计算: 使用者视角, 两个指针的大小的整数倍(不包括下一个块的prevsize), 或者指针大小的奇数倍(包括下一个块的prevsize). 采用后一种说法

管理者视角, 每个堆块前面有size和prevsize, 其中prevsize属于前一个堆块, 当前一个堆块是空闲的时候, 会放上前一个堆块的大小. (有没有标志位?? TODO). 管理者视角来说的话, 堆块的大小为: (n \* 两个指针的大小) + 指针大小(prevsize) + 指针大小(size). 也就是n+1倍的两个指针大小. size域保存的就是这种大小. 因此谈到各种bin的时候也是指包括size域的大小.

chunk指针一般指向prev_size域的开始处. 

堆块siza域最低位是AMP. (32位的时候只有3bit, 但是64位的时候就有4bit没有用了. 但还是只用3bit) 

总结NON_MAIN_ARENA块和mmapped块与其他正常块的区别. 在libc_malloc调用int_malloc返回的时候, 会检测得到的堆块是不是当前arena的. ?? TODO

mmapped的块指一页内存大小的整数倍的分配来的内存. 其他两个bit会被忽略, 因为它是单独的一块, 不会和其他空闲块相邻, 也不会在任何arena里. 回收的时候会直接调用munmap

## malloc_state
malloc_state描述arena的结构体. 主线程的arena是全局变量, 其他的arena在堆上(TODO). non_main_arena 可以有多个"堆"(heap_info).
```c
struct malloc_state
{
  /* Serialize access.  */
  __libc_lock_define (, mutex);
  /* Flags (formerly in max_fast).  */
  int flags;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];
  /* Base of the topmost chunk 不在其他任何bin里 */
  mchunkptr top;
  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;
  /* Unsorted, small and large bins */
  mchunkptr bins[NBINS * 2 - 2];

  /* Bitmap of bins 表示某个bin空 */
  unsigned int binmap[BINMAPSIZE];

  /* Linked list, 组织各个arena */
  struct malloc_state *next;
  /* Linked list for free arenas.  Access to this field is serialized
     by free_list_lock in arena.c.  */
  struct malloc_state *next_free;
  /* Number of threads attached to this arena.  0 if the arena is on
     the free list.  Access to this field is serialized by
     free_list_lock in arena.c.  */

  INTERNAL_SIZE_T attached_threads;
  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};

typedef struct malloc_state *mstate;
```

## bins
fastbin有10个,位于fastbinsY, 单链表, 栈式后进先出, 大小是 `(1 * 两个指针的大小) + 2 * 指针大小` 到 `(10 * 两个指针的大小) + 2 * 指针大小`. 内部的堆块标记为使用中, 不前后合并

其他的bin都是双链表.`mchunkptr bins[NBINS * 2 - 2];`中, 两个指针是一个bin. 下标为0的bin没有被使用, 下标为1的是unsorted bin. 下标2-63的是small bin. 下标64-126的是large bin.

small bins 有 62个. 列表式的先进后出. 范围是16=0x10 --- 504=0x1f8大小.(含header) 64位是32=0x20 - 1008=0x3f0大小

large bins 有63个. 前32个, 每个bin管理64大小, 后16个, 每个bin管理512字节的范围, 8个4096, 2个262144, 1个剩下的任何大小.

top chunk是最底下的chunk, 使用sbrk的时候扩大的就是这个chunk. 它的prev_inuse位总是在的, 因为相邻的free chunk在free的时候总会被合并.

last remainder chunk 上一个被分隔的chunk


## bins的循环

综述: free的bins首先放到unsorted里, malloc遍历unsorted的时候顺便整理放到各个bins里

### malloc_init_state 

对非fastbin, 创建头节点指向尾节点的循环
设置mstate的flags中的FASTCHUNKS_BIT.
初始化top chunk为第一个unsorted bin中的chunk.

### _int_malloc
__libc_malloc获取了arena后调用该函数.
如果大小在fastbin中. 去fastbin中找, 没有则到下一步, 有则检查得到的块, 检查通过后返回.
大小在small bin的时候, 去small bin中找, 如果对应的bin为空, 则下一步. 有则从末尾取一个, 检查一下另外一个方向的链表是否正常. 然后设置内存相邻的下一个chunk的prev_inuse, 最后返回
大小在large bin的, 也到large bin里找. 找完后调用malloc_consolidate (如果arena有FASTCHUNKS_BIT).
如果都没找到就遍历unsorted bin, (只有这时候才会把chunk放到bins里面) 从尾部遍历, 遍历的时候插入large bin的时候会总是插入第二个位置.
当1. 申请的chunk是small bin大小. 2. 当前的chunk是last remainder. 3. 这个chunk的大小大于请求的大小, 则将这个chunk分割, 剩下的部分还放回unsorted bin.
还没找到的话, 如果是large bin, 就遍历每个更大的large bin, 找到小的但大于要求大小的large bin. 能分隔则分隔, 不能分隔(剩下的空间小于最小chunk大小)则不分隔返回. 分隔出来的chunk插入到unsorted bin 末尾.
如果是small bin, 开始考虑更大small bin的分割. 同样找到最小的但大于要求大小的chunk分隔.
如果都不能满足, 则使用top chunk. 剩下的成为新的top chunk
如果还不能满足, 调用sysmalloc用mmap分配内存.

## _int_free

如果在fastbin 区间内, 插入人fastbin
再前后合并, 注意和top chunk的合并, 检查unsorted bin 并插入头节点.


### malloc_consolidate
遍历每个fastbin, 前后如果有free chunk先调用unlink后合并, 放到unsorted bin 头部里面去.
如果是top chunk当然和top chunk 合并

## 层次化描述malloc
malloc和free这内存管理的逻辑过于复杂, 而且很多逻辑耦合比较紧密, 不好拆开分块理解. 导致了学习的难度. 这里试图采取迭代的思想, 毕竟大型项目都是从简单到复杂的迭代出来的.

### small bin+large bin模型
该模型中只有small bin和large bin. 堆块的分配, 第一阶段是精确查找. 无法在对应的bin中找到时进入第二阶段是best fit查找, 找到满足要求的最小的堆块, 分隔或者不分割得到最终的堆块. 
free的时候也前后合并, 合并了再放到bin里. 
该模型还包含了top chunk. free的时候如果和top chunk 相邻, 则和top chunk合并. 
当small/large bin中任何chunk都无法满足的时候, 首先看top chunk, 然后使用mmap去满足.
包含了binmap的数据结构, 方便跳过空的bins. binmap中标记为空的bin一定为空, 但是标记为有的bin则不一定必须有chunk, 也可以为空.

(精确查找阶段对于large bin是只要求处于相同bin内还是必须相同大小?? TODO 怀疑是后者)

### sl(small large) + unsorted bin模型
该模型加入了unsorted bin. free的时候直接放入unsorted bin开头, 而malloc的时候, 在精确查找和best fit查找之间插入unsorted bin查找, 在末尾一边找一边处理unsorted bin.
当unsorted bin碰到大小合适的bin的时候直接返回, 否则就一直查找处理(把遍历过的chunk插入合适的small/large bin中).

为了使unsorted bin处理的时间更加均匀, 处理unsorted bin中的chunk最多处理MAX_ITER个.

### 改进1 减少多次分割时的开销.
经常会碰到小bin完全空, 分配时总是去某个large bin中分割的情况. 这种情况每次分配小块的时候都需要遍历一次很多small bin和large bin. 可以做出改进.
当每次有split的时候, 将剩下的chunk作为last remainder chunk单独指针保存, 并且插入unsorted bin的末尾. 
当遍历unsorted bin的时候, 如果是小chunk(在small bin范围内), 当前指向的chunk是last remainder chunk, 并且大小大于要求的大小, 则优先分隔该chunk直接返回.

### slu + fast bin模型
增加10个fast bin 作为上述模型的外包层. free的时候, 如果是fast bin范围内的直接放入fast bin(因为fast bin无限容量.233 这也说明unsorted bin不会有fast bin范围的chunk?? TODO) malloc的时候的精确查找阶段先去fast bin里面找(fast bin范围内), 没有再去small/large bin里找.

引入 malloc_consolidate函数, 用于把fast bin中的chunk清理到small bin 中去. 引入flags中的 FASTCHUNKS_BIT 指示当前的arena有没有fast bin.

best fit阶段也不能满足, 找到了topchunk. 如果top chunk也不能满足要求, 就先清理掉fast bin再去mmap. 调用malloc_consolidate, 然后再去重新遍历unsorted bin, 把所有的chunk都清理了. 之后再回到这里, 发现没有fast bin的时候再通过mmap满足要求.

当分配large bin而精确查找阶段也满足了的时候也调用 malloc_consolidate.


## sluf + tcache模型

在fast bin之前增加tcache. (在__libc_malloc()调用_int_malloc()之前)在获取arena之前, 就先看tcache, 有则直接返回. free的时候优先放到tcache, 满了才继续放到别处.

tcache是很多个链表, 保存大小相同的chunk.
tcache是直接指向下一个的tcache_next, 而不是指向堆块头部. 直接形成链表
tcache_perthread_struct用于维护各个tcache内空闲堆块的数量, 和索引各个tcache.
第一次malloc的时候, 会malloc一块区域保存tcache_perthread_struct.


## 与其他部分的关系

glibc2.26 开始有了tcache, 并默认开启. tcache比small bin还多一点. 
内存释放的时候, tcache没满优先放到tcache. 分配的时候, 调用malloc之前看看tcache有没有.
申请fastbin大小的内存的时候, 找到fastbin内如果找到, 把fastbin上其他块填入tcache中. smallbin同理.
处理unsorted bin的时候, 即使找到大小合适的块, 也不直接返回, 而是


## 查阅的资料

64位时, 默认开启的fastbin范围(chunk总大小)是0x20 - 0x80
32位TODO

tcache是64个单向链表，最多7个节点(chunk)，chunk的大小在32bit上是8到512（8byte递增）；在64bits上是16到1024（16bytes递增）。
fastbin只有10个链表, 范围肯定很小, 而和smallbins有62个, 大小基本重合.

当某一个tcache链表满了7个，再有对应的chunk（不属于fastbin的）被free，就直接进入了unsortedbin中。
tcache_perthread_struct结构，一般是在heapbase+0x10（0x8）的位置。对应tcache的数目是char类型。

## 待整理
> 绕过tcache使得堆块free后进入unsorted bin的方式通常有两种：

> 每个tcache链上默认最多包含7个块，再次free这个大小的堆块将会进入其他bin中，例如tcache_attack/libc-leak
> 默认情况下，tcache中的单链表个数是64个，64位下可容纳的最大内存块大小是1032（0x408），故只要申请一个size大于0x408的堆块，然后free即可
