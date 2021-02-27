---
layout:     post
title:      "JVM--内存管理"
date:       2021-02-21 20:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JVM

---



## 导航
[一. JVM运行时数据区](#jump1)
<br>
[二. 对象内存布局](#jump2)
<br>
[三. 对象存活判别](#jump3)
<br>
[四. JAVA引用类型](#jump4)
<br>
[五. 可达性分析中对象的生存死亡](#jump5)
<br>
[六. 分代收集](#jump6)
<br>
[七. 三色标记算法](#jump7)
<br>









<br><br>
## <span id="jump1">一. JVM运行时数据区</span>

[![6SvwdA.png](https://s3.ax1x.com/2021/02/27/6SvwdA.png)](https://imgtu.com/i/6SvwdA)



<br><br>
## <span id="jump2">二. 对象内存布局</span>

[![yT6ojS.png](https://s3.ax1x.com/2021/02/21/yT6ojS.png)](https://imgchr.com/i/yT6ojS)



<br><br>
## <span id="jump3">三. 对象存活判别</span>

[![y75kxs.png](https://s3.ax1x.com/2021/02/22/y75kxs.png)](https://imgchr.com/i/y75kxs)



<br><br>
## <span id="jump4">四. JAVA引用类型</span>

[![y7He4f.png](https://s3.ax1x.com/2021/02/22/y7He4f.png)](https://imgchr.com/i/y7He4f)



<br><br>
## <span id="jump5">五. 可达性分析中对象的生存死亡</span>

[![yL2Frq.png](https://s3.ax1x.com/2021/02/23/yL2Frq.png)](https://imgchr.com/i/yL2Frq)



<br><br>
## <span id="jump6">六. 分代收集</span>

[![6pleNn.png](https://s3.ax1x.com/2021/02/27/6pleNn.png)](https://imgtu.com/i/6pleNn)



<br><br>
## <span id="jump7">七. 三色标记算法</span>

三色标记具体指那三色
* 黑色: 根对象,或者该对象与它的子对象都被扫描过.黑色对象不可能直接(不经过灰色对象)指向某个白色对象
* 灰色: 对象本身已被扫描,但这个对象上至少存在一个引用还没有被扫描过
* 白色: 未被扫描的对象,如果扫描完所有对象之后,最终为白色的为不可达对象,即垃圾对象

引用链遍历分析与用户线程异步并发执行会造成的问题:
* 把原本消亡的对象错误标记为存活----产生浮动垃圾逃脱本次垃圾收集,但可接受,能在下次回收中清除
* 把原本存活的对象错误标记为消亡----致命问题,不能接受

下图演示把原本存活的对象错误标记为消亡的致命错误具体是如何产生的
[![6p83DS.png](https://s3.ax1x.com/2021/02/27/6p83DS.png)](https://imgtu.com/i/6p83DS)

当且仅当以下两个条件同时满足时,会产生"对象消失"的致命问题,即原本应该是黑色的对象被误标为白色:
* 赋值器插入了一条或多条从黑色对象到白色对象的新引用
* 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用

因此,要解决并发扫描时的对象消失问题,只需要破坏这两个条件的任意一个即可.由此分别产生了两种解决方案:
* 增量更新(Incremental Update): 当黑色对象插入新的指向白色对象的引用关系时,就将这个新插入的引用记录下来,等并发扫描结束后,再以这些记录过的引用关系中的黑色对象为根,重新扫描一次.这可以简化理解为:当一个白色对象被黑色对象引用,将黑色对象重新标记为灰色,让垃圾回收器重新扫描.<font color="red">CMS采用</font>
* 原始快照(Snapshot At Beginning,SATB): 当灰色对象要删除指向白色对象的引用关系时,就将这个要删除的引用记录下来,在并发扫描结束之后,再以这些记录过的引用关系中的灰色对象为根,重新扫描一次.总而言之就是:无论引用关系删除与否,都会按照刚开始扫描的那一刻的对象图快照来进行搜索.<font color="red">G1采用</font>

