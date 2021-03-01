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
[八. HotSpot虚拟机垃圾收集器](#jump8)








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

这里稍稍总结一下,在HotSpot的算法实现细节中,根节点枚举、安全点、安全区域、记忆集与卡表、写屏障这几部分,属于GC Roots分析部分;并发的可达性分析,属于根据GC Roots进行引用链遍历追踪分析部分.<br>

这正好囊括了可达性分析的两个部分:头和引用链.<br>

另外,在GC Roots分析的部分,涉及到Oop Map和记忆集两块辅助实现优化.Oop Map解决的是如何更**<font color="red">快</font>**的找到根节点的问题;记忆集解决的是如何更**<font color="red">全</font>**的找到根节点的问题.



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

为啥G1不使用增量更新算法呢?<br>

因为使用增量更新算法,那变成灰色的对象还要重新扫描一遍,效率太低了,所以G1在处理并发标记的过程比CMS效率要高,这个主要是解决漏标的算法决定的.<br>



<br><br>
## <span id="jump8">八. HotSpot虚拟机垃圾收集器</span>

垃圾收集算法是内存回收的方法论,而垃圾收集器是内存回收的实践者.JVM规范中对垃圾收集器应该如何实现并没有做出任何规定,因此不同厂商不同版本的虚拟机所包含的垃圾收集器可能会有很大差别,不同虚拟机一般也都会提供各种参数供用户根据自己的应用特点和要求组合出各个内存分代所使用的收集器.<br>

这里分析HotSpot虚拟机中的垃圾收集器.<br>

[![69qtSO.png](https://s3.ax1x.com/2021/02/28/69qtSO.png)](https://imgtu.com/i/69qtSO)

这个关系不是一成不变的,由于维护和兼容性测试的成本,在JDK8时将Serial+CMS、ParNew+Serial Old这两个组合生命为废弃,并在JDK9中完全取消了这些组合的支持.<br>


<br>
**<font size="4">Serial</font>**<br>

[![69XKOg.png](https://s3.ax1x.com/2021/02/28/69XKOg.png)](https://imgtu.com/i/69XKOg)


<br>
**<font size="4">Serial Old</font>**<br>

Serial收集器的老年代版本,也可作为CMS收集器发生失败时的后备预案,在并发收集发生Concurrent Mode Failure时使用.
[![69XKOg.png](https://s3.ax1x.com/2021/02/28/69XKOg.png)](https://imgtu.com/i/69XKOg)


<br>
**<font size="4">ParNew</font>**<br>

[![69XLjS.png](https://s3.ax1x.com/2021/02/28/69XLjS.png)](https://imgtu.com/i/69XLjS)

Serial的并行版本.<br>

若老年代采用CMS,则新生代只能采用ParNew搭配.<br>


<br>
**<font size="4">Parallel Scavenge</font>**<br>

新生代收集器,同样是基于标记-复制算法实现的收集器,也是能够并行收集的多线程收集器.<br>

Parallel Scavenge收集器的目标是打到一个可控的吞吐量.吞吐量优先的收集器<br>
``
吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 运行垃圾收集时间 )
``

CMS等收集器的关注点是尽可能的缩短垃圾收集时用户线程的停顿时间.


<br>
**<font size="4">Parallel Old</font>**<br>

Parallel Old是Parallel Scavenge收集器的老年代版本,支持多线程并发收集,基于标记-整理算法实现.
[![6CpN11.png](https://s3.ax1x.com/2021/02/28/6CpN11.png)](https://imgtu.com/i/6CpN11)


<br>
**<font size="4">CMS 收集器</font>**

CMS(Concurrent Mark Sweep)收集器是一种以获得最短回收停顿时间为目标的收集器.基于标记-清除算法实现,运作过程分四个阶段:
* 初始标记----只标记一下GC Roots能直接关联到的对象,速度很快,可以理解为就是前面讲的"根节点枚举",需要STW
* 并发标记----从GC Roots的直接关联对象开始遍历整个对象图的过程,耗时较长,但不需要停顿用户线程(即STW),可以与垃圾收集线程一起并发运行.这得益于三色标记算法,CMS采用的是增量更新方案.
* 重新标记----为了修正并发标记期间,因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录,耗时比初始标记稍长但远小于并发标记,需要STW.CMS采用的是增量更新方案.
* 并发清除----清理删除掉标记阶段判断的已死亡对象,由于不需要移动存活对象,所以这个阶段也可以与用户线程并发.

[![6CArrR.png](https://s3.ax1x.com/2021/02/28/6CArrR.png)](https://imgtu.com/i/6CArrR)

存在缺点:<br>

**缺点一**<br>
由于CMS无法处理"浮动垃圾",有可能会出现"Concurrent Mode Failure"失败而导致另一次完全STW的Full GC的产生.原因如下<br>

在CMS并发标记和并发清除阶段,用户线程是还在继续运行的,因此就会伴随着有新的垃圾对象不断产出,但这部分垃圾对象是出现在标记过程结束以后,CMS无法在当次收集中处理掉它们,只能留待下一次垃圾收集时再清理.这部分垃圾就称为"浮动垃圾".同样也是由于在垃圾收集阶段用户线程还需要持续运行,那就还需要预留足够内存空间提供给用户线程使用,因此CMS收集器不能像其他收集器那样等待到老年代几乎完全被填满了再进行收集,必须预留一部分空间供并发收集时的程序运作使用.可通过"-XX:CMSInitiatingOccupancyFraction"来设置.<br>

设置的小会导致回收频率高,降低服务进程性能.设置的大又会面临另一种风险:CMS运行期间预留的内存无法满足程序分配新对象的需要,就会出现一次"并发失败"(Concurrent Mode Failure),这时候虚拟机将不得不启动后备预案:冻结用户线程的执行,临时启用Serial Old收集器来重新进行老年代的垃圾收集,但这样停顿时间就很长了.所以参数"-XX:CMSInitiatingOccupancyFraction"设置得太高将会很容易导致大量的并发失败产生,性能反而降低,用户应在生产环境中根据实际应用情况来权衡设置.<br>

**缺点二**<br>
CMS基于标记-清除算法实现,会产生内存碎片.空间碎片过多时,会给大对象分配带来很大麻烦,往往会出现老年代还有很多剩余空间,但就是无法找到足够大的连续空间来分配当前对象,而不得不提前触发一次Full GC的情况.为了解决这个问题,CMS收集器提供了一个-XX:+UseCMSCompactAtFullCollection开关参数(默认开启,此参数从JDK9开始废弃),用于在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程,由于这个内存整理过程必须移动存活对象,(在shenandoah和ZGC出现前)是无法并发的.这样空间碎片问题是解决了,但停顿时间又会变长,因此虚拟机提供了另外一个参数:"-XX:CMSFullGCsBeforeCompaction"(JDK9开始废弃),这个参数的作用是要求CMS收集器在执行过若干次(数量由参数值决定)不整理空间的Full GC之后,下一次进入Full GC前会先进行碎片整理(默认0,表示每次进入Full GC时都进行碎片整理),也就是MSC,MSC的全称是Mark Sweep Compact,即标记-清理-压缩,MSC是一种算法,请注意Compact,即它会压缩整理堆,这一点很重要<br>

注意这里"-XX:CMSFullGCsBeforeCompaction"标记的是Full GC次数,而不是CMS GC次数,这个要区分清.CMS GC是针对老年代的GC,不同于Full GC.而老年采用CMS收集器时,如果发生了需要进行Full GC的情况,这时是会退化到后备预案启用Serial Old来进行的,也就是单线程,全程STW.<br>


<br>
**<font size="4">Garbage First收集器</font>**<br>

以停顿时间可控为目标的收集器,面向局部收集的设计思路和基于Region的内存布局形式.收集整体是使用"标记-整理",Region之间基于"复制"算法,GC后会将存活对象复制到可用分区(未分配的分区),所以不会产生空间碎片.<br>

G1出现之前的所有收集器,包括CMS在内,垃圾收集的目标范围要么是整个新生代(Minor GC),要么就是整个老年代(Major GC),再要么就是整个Java堆(Full GC,当然Full GC包括了方法区).G1突破了这个限制,它可以面向堆内存任何部分来组成回收集(Collection Set,一般简称CSet)进行回收,衡量标准不再是它属于哪个分代,而是哪块内存中存放的垃圾数量最多,回收收益最大,这就是G1收集器的**<font color="red">Mixed GC模式</font>**<br>

