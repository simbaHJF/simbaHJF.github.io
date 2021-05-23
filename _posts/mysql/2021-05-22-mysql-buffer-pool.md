---
layout:     post
title:      "mysql -- buffer pool"
date:       2021-05-22 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---






## 导航
[一. mysql的buffer pool设计结构](#jump1)
<br>
[二. 简单LRU链表在Buffer Pool实际运行中的问题](#jump2)
<br>
[三. 基于冷热数据分离的思想设计LRU链表](#jump3)
<br>
[四. 脏页刷盘机制](#jump4)
<br>
[五. 更新机制流程](#jump5)
<br>








<br><br>
## <span id="jump1">一. mysql buffer pool的基础设计结构</span>

[![gqLPQf.png](https://z3.ax1x.com/2021/05/22/gqLPQf.png)](https://imgtu.com/i/gqLPQf)


另外,这里还有一个问题就是:怎么知道一个数据页有没有被缓存呢?<br>

这里还有一个哈希表数据结构,mysql会用'表空间号 + 数据页号',作为一个key,然后缓存页的地址作为value.<br>

当要使用一个数据页的时候,通过'表空间号 + 数据页号'作为key,去这个哈希表里查一下,如果没有就读取数据页,如果已经有了,就说明数据页已经被缓存了.<br>



<br><br>
## <span id="jump2">二. 简单LRU链表在Buffer Pool实际运行中的问题</span>

<br>
**<font size="4">预读机制带来的问题</font>** <br>

MySQL的预读机制为:当从磁盘上加载一个数据页的时候,它可能会连带着把这个数据页相邻的其他数据页,也加载到缓存里去.但是接下来实际只有一个缓存页是被访问了,另外一个通过预读机制加载的缓存页其实并没有人访问,此时这两个缓存页可能都在LRU链表的前面.<br>


<br>
**<font size="4">全表扫描遍历</font>** <br>

全表扫描遍历也会导致把该表的数据一一装入LRU链表头部,但后续可能对这些数据的访问就没有了.<br>



<br><br>
## <span id="jump3">三. 基于冷热数据分离的思想设计LRU链表</span>

出于对上述问题的考虑,真正MySQL在设计LRU链表的时候,采取的实际上是冷热数据分离的思想.<br>

真正的LRU链表,会被拆分为两个部分,一部分是热数据,一部分是冷数据,即冷热LRU.这个冷数据的比例是由innodb_old_blocks_pct参数控制的,默认是37,也就是说冷数据占比37%.<br>


<br>
**<font size="4">冷热分离LRU链表设计下的数据加载机制</font>**<br>

1. 首次加载数据时,缓存页会被放在冷数据区域(冷LRU)的链表头部
2. 一个数据页被加载到冷LRU缓存链表头部后,在1s之后再次访问这个数据页,它会被挪到热数据区域(热LRU)的链表头部去,这个时长由参数innodb_old_blocks_time参数控制,默认1000ms.
	> 这种设计是因为:当加载了一个数据页到缓存中后,过了1s后还会再次访问到这个缓存页,说明后续很可能会经常要访问它,此时才会把它放到热LRU头部

[![gLpldK.png](https://z3.ax1x.com/2021/05/22/gLpldK.png)](https://imgtu.com/i/gLpldK)

此时,如果内存页不够用了需要淘汰一部分缓存时,优先会选择冷数据区域的尾部的缓存页.<br>


<br>
**<font size="4">热LRU链表的一个优化</font>** <br>

热LRU链表的数据是会被经常访问的,因此经常移动其缓存页的位置也对性能会有一定的影响,因此这里有一个优化点就是:<font color="red">只有在热数据区的后3/4的缓存页被访问了,才会将其移动到热LRU链表头部,如果是热LRU链表的头部1/4缓存页被访问,它是不会被移动到链表头部的.</font> <br>



<br><br>
## <span id="jump4">四. 缓存页淘汰策略</span>

<br>
**<font size="4">缓存页满 + 定时刷盘</font>** <br>

缓存页满时会主动触发将冷LRU尾部数据淘汰的机制,另外会有一个后台线程定时的被动执行冷LRU尾部数据淘汰的工作,淘汰后,会清空缓存页,并将其归还给free链表.<font color="red">当然,淘汰缓存数据时,如果发现其正好在脏页链表中(flush链表),会先完成对脏页的刷盘,之后才会释放并规划给free链表</font>


<br>
**<font size="4">flush链表定时刷盘</font>** <br>

只把冷LRU链表缓存页回收刷盘还并不够,LRU链表的热数据区里的很多缓存页可能会被频繁的修改,因此也需要适当的时机,在MySQL并不频繁的时候,把flush链表的缓存页刷入磁盘,然后将其从flush链表和lru链表中移除,归还给free链表.<br>



<br><br>
## <span id="jump5">五. 更新机制流程</span>

[![gL3SqP.png](https://z3.ax1x.com/2021/05/22/gL3SqP.png)](https://imgtu.com/i/gL3SqP)