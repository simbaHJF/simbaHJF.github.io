---
layout:     post
title:      "Cache Line"
date:       2021-05-13 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 计算机组成原理

---








## 导航
[一. 什么是Cache Line](#jump1)
<br>
[二. Cache 的数据结构和读取过程](#jump2)
<br>
[三. CPU从Cache读取数据过程](#jump3)
<br>
[四. Cache数据更新过程](#jump4)
<br>









<br><br>
## <span id="jump1">一. 什么是Cache Line</span>

CPU从内存中读取数据到CPU Cache(L1,L2,L3缓存)的过程中,是一小块一小块来读取数据的,这样一小块一小块的数据就是Cache与DRAM(内存)交互的最小单位,在CPU Cache里面,被称作Cache Line(缓存行),通常为 64 Byte.当 CPU 把内存的数据载入 cache 时,会把临近的共 64 Byte 的数据一同放入同一个Cache line,因为空间局部性:临近的数据在将来被访问的可能性大.



<br><br>
## <span id="jump2">二. Cache 的数据结构和读取过程</span>

现代 CPU 进行数据读取的时候,无论数据是否已经存储在 Cache 中，CPU 始终会首先访问 Cache.只有当 CPU 在 Cache 中找不到数据的时候,才会去访问内存,并将读取到的数据写入 Cache 之中

[![gBtNy8.png](https://z3.ax1x.com/2021/05/13/gBtNy8.png)](https://imgtu.com/i/gBtNy8)



<br><br>
## <span id="jump3">三. CPU从Cache读取数据过程</span>

[![gBUa5j.png](https://z3.ax1x.com/2021/05/13/gBUa5j.png)](https://imgtu.com/i/gBUa5j)

内存地址对应到 Cache 里的数据结构,多了一个有效位和对应的数据,由"索引 + 有效位 + 组标记 + 数据"组成.<br>

如果内存中的数据已经在 CPU Cache 里了,那一个内存地址的访问,就会经历这样 4 个步骤:
1. 根据内存地址的低位,计算在 Cache 中的索引
2. 判断有效位,确认 Cache 中的数据是有效的
3. 对比内存访问地址的高位,和 Cache 中的组标记,确认 Cache 中的数据就是我们要访问的内存数据,从 Cache Line 中读取到对应的数据块(Data Block)
4. 根据内存地址的 Offset 位,从 Data Block 中,读取希望读取到的字

如果在 2、3 这两个步骤中,CPU 发现,Cache 中的数据并不是要访问的内存地址的数据,那 CPU 就会访问内存,并把对应的 Block Data 更新到 Cache Line 中,同时更新对应的有效位和组标记的数据.<br>

这里再解释一下步骤1,CPU为了知道内存数据存放在Cache中的哪个位置,采用'直接映射 Cache'的方式.直接映射 Cache 采用的策略,就是确保任何一个内存块的地址,始终映射到一个固定的 CPU Cache 地址(Cache Line).举例来讲,如共有8个缓存行,当前要访问第21号内存块,若其在Cache Line中,那它一定在5(21 mod 8 = 5)号Cache Line中.<br>



<br><br>
## <span id="jump4">四. Cache数据更新</span>

<br>
**<font size="4">写直达(Write-Through)策略</font>**<br>

[![gBq4MR.md.jpg](https://z3.ax1x.com/2021/05/13/gBq4MR.md.jpg)](https://imgtu.com/i/gBq4MR)
写直达策略的问题是: 太慢!!!!<br>

<font color="red">因此,现在硬件采用的都是下面的写回策略</font> <br>


<br>
**<font size="4">写回(Write-Back)</font>** <br>

[![gBI1KK.md.jpg](https://z3.ax1x.com/2021/05/13/gBI1KK.md.jpg)](https://imgtu.com/i/gBI1KK)
