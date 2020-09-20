---
layout:            post
title:             "数据库缓存管理器块替换"
date:              2017-12-25 18:25:00 +0300
menutitle:         "数据库缓存管理器块替换"
category:          Features
author:            tangoo
tags:              DB
comments:          true
---

学习高级数据库技术课的时候，老师提到了数据库中的buffer manager(缓存管理器)，现在找出来重新整理温习下，今天要讨论的数据库缓存管理器块替换的总体结构如下图所示：
<figure>
   <img src='{{ "/media/img/2017/higher-level-access.gif" | absolute_url }}' />
   <figcaption>HIGHER LEVEL ACCESS.</figcaption>
</figure>

当应用程序需要从磁盘上得到一个块时，它会调用缓冲管理器，那么现在就分两种情况：

1. 如果这个块已经在buffer中，缓冲管理器就会返回该块在主存中的地址给应用程序
2. 如果这个块不在buffer中，那么buffer manager就会做以下几件事情:
    - 在buffer中为该块分配空间，那么它是如何做到的呢？首先，如果需要的话，替换一些块以让出空间给新的块，被替换的块如果已经被修改过，那么就需要把它重新写回到磁盘上。
    - 把需要的块从磁盘上读到buffer中，然后返回该块在主存中的地址。

## 第二部分
在上面的述说中我们了解到，当buffer的空间不足时，会选择一个块换出，这里就牵涉到了替换strategy，记得老师在课上给我们讲解了几种比较流行的替换strategy，自己整理了一下:
### 2.1、LRU算法(Least Recently Used 近期最少使用算法)
这个应该非常熟悉，不光是在数据库中用到，我觉得我们最早接触到这个应该是在操作系统中，这里就不多说了，给个example就很明白了。
假设现在的访问序列：1，4，8，1，5，2，3，2，4
得到如下图所示的替换过程：
<figure>
   <img src='{{ "/media/img/2017/access.gif" | absolute_url }}' />
   <figcaption>ACCESS.</figcaption>
</figure>

但是LRU算法在一些特定的情况下，性能就不会很好，比如：
- 文件共享：马上被用到的数据被一次访问的数据挤出，LRU堆栈保留了一次性访问的数据直到堆栈满的时候才会被替换出；
- 环访问：假设现在LRU stack的大小为k，而现在的访问序列是：1，…，k，k+1，那么顺序访问k+1之后，就会一直miss(之前的元素已被替换出堆栈)；
- 不同频率的访问(混合工作负载)；
频繁访问的数据会被不频繁访问的数据所替换。

### 2.2、Clock: An approximation of LRU(一种与LRU近似的方法)
该算法的思想是把所有的块形成一个环(通过mod运算实现)，环中的每一块都带一个引用位(reference_bit)，也称为第二次机会位，当可用块为0时，此时就需要选择一块换出，在选择块换出时，如果当前块的reference_bit为1时，则修改成0，该块继续留在环中，如果当前块的reference_bit为0，则选择该块换出，如下图所示：
<figure>
   <img src='{{ "/media/img/2017/replacement.gif" | absolute_url }}' />
   <figcaption>REPLACEMENT.</figcaption>
</figure>￼

### 2.3、LIRS Policy
#### 2.3.1、LIRS算法
(Low Inter-Reference Recency Set Recency)：表示某一块最近一次被访问与当前的间隔，可以理解为该块最近一次被访问之后，接下来访问了多少个其它块(Recency可以用这个其它块数表示)。
Reuse Distance(IRR)：Buffer中某一块被连续访问时，它们之间的其它块数(在计算的时候，不算重复的块)。可以用下图进行说明：
<figure>
   <img src='{{ "/media/img/2017/irr.gif" | absolute_url }}' />
   <figcaption>IRR.</figcaption>
</figure>￼￼

#### 2.3.2、LIRS的基本思想
- 该算法认为IRR值高的块不会被频繁访问，所以把这些块选作将要被替换的块；
- Recency被用作第二次引用；
- 把IRR值低的块保留在Buffer Cache中。
LIRS对比上面两种算法有其明显的改进，首先，它根据多种信息资源来合理地改变每个块的状态；其次，它的实现成本低。

#### 2.3.3、LIRS的数据结构
LIRS的整个数据结构如下图所示：
<figure>
   <img src='{{ "/media/img/2017/lirs.gif" | absolute_url }}' />
   <figcaption>LIRS.</figcaption>
</figure>￼

我们从上图中可以看出，该算法有两个block set：LIR block set和HIR block set，同时整个物理缓存也由两部分组成，并且也可以知道，属于IRR值低的块集合中的块全部保存在物理缓存中，而属于IRR值高的块集合中的块只有一部分位于物理缓存中。

#### 2.3.4 LIRS的块替换操作
下面我们结合具体的例子来了解使用LIRS算法之后，是如何进行块替换的，假设现在有5个块，分别是A,B,C,D,E，且当前在time 9这个位置，此时块访问序列以及每个块的Recency和IRR值如下图所示：
<figure>
   <img src='{{ "/media/img/2017/lir-hir.gif" | absolute_url }}' />
   <figcaption>LIR AND HIR.</figcaption>
</figure>￼
￼
由上图可知，块A最近一次被访问是在time 8，所以它的R（Recency）为1；另外块A在time 6 和time 8被连续两次访问到，在这两个时刻之间，只有块D被访问到，所以块A的IRR为1。块B,C,D,E的R值和IRR值的计算方法和块A一样，如果某一块最近只有一次被访问到，那么它的IRR值为无穷大，比如块C和E。
假设现在的物理缓存的空间划分如下图所示：
<aside>
<figure class="right">
<img src='{{ "/media/img/2017/lirs-hirs.gif" | absolute_url }}' />
<figcaption>LIRS AND HIRS.</figcaption>
</figure>
</aside>

也就是，在该物理缓存中只能存放两个IRR值低的块，一个IRR值高的块，我们最后进行归类得到如下图所示：
<figure>
   <img src='{{ "/media/img/2017/lir-block.gif" | absolute_url }}' />
   <figcaption>LIR BLOCK.</figcaption>
</figure>￼

那么接下来，如果在time 10访问块D，该如何进行替换，从上面得知，块D属于IRR值高的块集合，而此时属于IRR值高的块集合中只有块E驻留在物理缓存中，那么此时就需要把块E替换出去，把块D存入物理缓存中，整个过程如下图所示：
<figure>
   <img src='{{ "/media/img/2017/time-block.gif" | absolute_url }}' />
   <figcaption>TIME BLOCK.</figcaption>
</figure>￼
￼
在块D进入物理缓存之后，我们接下来要做一些更新操作，看看块D应该属于哪个集合，因为在D进入物理缓存之后，各个块的R值和IRR值已经发生变化，更新完各个块的R值和IRR值之后，我们把块D的IRR值和IRR值低的块集合中的块的R值进行比较，如果块D的IRR值比IRR值低的块集合中的某块的R值小，则把该块换入到IRR值高的块集合中，而块D进入IRR值低的块集合中，过程图如下图所示：
<figure>
   <img src='{{ "/media/img/2017/time-blocks.gif" | absolute_url }}' />
   <figcaption>TIME BLOCKS.</figcaption>
</figure>￼￼
块D在time 10被访问之后，最后的结果如下图所示：
<figure>
   <img src='{{ "/media/img/2017/time-blocks2.gif" | absolute_url }}' />
   <figcaption>TIME BLOCKS.</figcaption>
</figure>