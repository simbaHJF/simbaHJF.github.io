---
layout:     post
title:      "JVM--内存管理"
date:       2021-03-01 20:30:00 +0800
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
<br>
[九. 低延迟垃圾垃圾收集器](#jump9)
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

这里稍稍总结一下,在HotSpot的算法实现细节中,根节点枚举、安全点、安全区域、记忆集与卡表、写屏障这几部分,属于GC Roots分析部分;并发的可达性分析,属于根据GC Roots进行引用链遍历追踪分析部分.<br>

这正好囊括了可达性分析的两个部分:头和引用链.<br>

另外,在GC Roots分析的部分,涉及到Oop Map和记忆集两块辅助实现优化.Oop Map解决的是如何更**<font color="red">快</font>**的找到根节点的问题;记忆集解决的是如何更**<font color="red">全</font>**的找到根节点的问题,当然RSet也避免了全堆扫描跨代引用,因此也是一种更**<font color="red">快</font>**方面的优化



<br><br>
## <span id="jump7">七. 三色标记算法</span>

三色标记具体指那三色
* 黑色: 根对象,或者该对象与它的子对象都被扫描过.黑色对象不可能直接(不经过灰色对象)指向某个白色对象
* 灰色: 对象本身已被扫描,但这个对象上至少存在一个引用还没有被扫描过
* 白色: 未被扫描的对象,如果扫描完所有对象之后,最终为白色的为不可达对象,即垃圾对象

引用链遍历分析与用户线程异步并发执行会造成的问题:
* 把原本消亡的对象错误标记为存活 ----- 产生浮动垃圾逃脱本次垃圾收集,但可接受,能在下次回收中清除
* 把原本存活的对象错误标记为消亡 ----- 致命问题,不能接受

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
**<font size="5">Serial</font>**<br>

[![69XKOg.png](https://s3.ax1x.com/2021/02/28/69XKOg.png)](https://imgtu.com/i/69XKOg)


<br>
**<font size="5">Serial Old</font>**<br>

Serial收集器的老年代版本,也可作为CMS收集器发生失败时的后备预案,在并发收集发生Concurrent Mode Failure时使用.
[![69XKOg.png](https://s3.ax1x.com/2021/02/28/69XKOg.png)](https://imgtu.com/i/69XKOg)


<br>
**<font size="5">ParNew</font>**<br>

[![69XLjS.png](https://s3.ax1x.com/2021/02/28/69XLjS.png)](https://imgtu.com/i/69XLjS)

Serial的并行版本.<br>

若老年代采用CMS,则新生代只能采用ParNew搭配.<br>


<br>
**<font size="5">Parallel Scavenge</font>**<br>

新生代收集器,同样是基于标记-复制算法实现的收集器,也是能够并行收集的多线程收集器.<br>

Parallel Scavenge收集器的目标是打到一个可控的吞吐量.吞吐量优先的收集器<br>
``
吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 运行垃圾收集时间 )
``

CMS等收集器的关注点是尽可能的缩短垃圾收集时用户线程的停顿时间.


<br>
**<font size="5">Parallel Old</font>**<br>

Parallel Old是Parallel Scavenge收集器的老年代版本,支持多线程并发收集,基于标记-整理算法实现.
[![6CpN11.png](https://s3.ax1x.com/2021/02/28/6CpN11.png)](https://imgtu.com/i/6CpN11)


<br>
**<font size="5">CMS 收集器</font>**

CMS(Concurrent Mark Sweep)收集器是一种以获得最短回收停顿时间为目标的收集器.基于标记-清除算法实现,运作过程分四个阶段:
* 初始标记 ----- 只标记一下GC Roots能直接关联到的对象,速度很快,可以理解为就是前面讲的"根节点枚举",需要STW
* 并发标记 ----- 从GC Roots的直接关联对象开始遍历整个对象图的过程,耗时较长,但不需要停顿用户线程(即STW),可以与垃圾收集线程一起并发运行.这得益于三色标记算法,CMS采用的是增量更新方案.
* 重新标记 ----- 为了修正并发标记期间,因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录,耗时比初始标记稍长但远小于并发标记,需要STW.CMS采用的是增量更新方案.
* 并发清除 ----- 清理删除掉标记阶段判断的已死亡对象,由于不需要移动存活对象,所以这个阶段也可以与用户线程并发.

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
**<font size="5">Garbage First收集器</font>**<br>

在分析G1之前,这里首先来说说为什么会出现G1,这对理解G1还是有帮助的.<br>

出现G1的原因,当然是因为CMS不够好了呗,要是那么好,就一直用它就好了嘛.那问题出在哪呢?由CMS的机制知道,CMS在老年代GC时,是整代GC,而随着技术发展,现在应用的堆占用越来越大,因此CMS在进行整代收集时,停顿时间就会随着堆大小增加而变长,这就是G1出现的原因,我们需要一个停顿时间可控的垃圾收集器.<br>

G1就是以停顿时间可控为目标的收集器,面向局部收集的设计思路和基于Region的内存布局形式.收集整体是使用"标记-整理",Region之间基于"复制"算法,GC后会将存活对象复制到可用分区(未分配的分区),所以不会产生空间碎片.<br>

Region:G1将内存划分成了多个大小相等的独立区域(Region),Region内存地址可以不连续.每个Region可根据需要,扮演以下几种角色之一:
* Eden
* Survior
* Old
* Humongous ----- 专门用来存储大对象.G1认为只要大小超过了一个Region容量一半的对象即可判定为大对象.G1大多数行为会将该区作为老年代的一部分来进行看待.

**<font color="red">特别说明: 某个region的类型不是固定的,比如一次ygc过后,原来的Eden的分区就会变成空闲的可用分区,随后也可能被用作分配巨型对象,也即用作Humongous区</font>**

G1 GC 分类:
* Minor GC / Young GC ----- 回收所有年轻代的Region.
	* 触发条件:当Eden区不能再分配新的对象并且无法申请足够内存就会触发
* Mixed GC ----- 回收所有的年轻代的Region + 部分老年代的Region.
	* 触发条件:一次YoungGc之后,老年代占据堆内存(包括old+humongous)的百分占比超过InitiatingHeapOccupancyPercent(默认45%)时,超过这个值就会触发Mixed GC.
	* 为何是部分老年代的Region: 通过"停顿预测模型"来预测本次收集的Region,对回收收益高的的Region进行回收以贴近用户设定的目标停顿时间.
* Full GC ----- G1的垃圾回收过程是和应用程序并发执行的,当Mixed GC的速度赶不上应用程序申请内存的速度的时候,Mixed G1就会降级到Full GC.开始版本Full GC使用的是单线程的Serial Old模式,会导致长时间的STW,JDK10以后,Full GC已经是并行运行,但是仍然要避免Full GC,注意这里并行是指回收线程并行,但仍不能与用户线程同时并发执行.


G1的几个关键问题:
* G1的内存占用问题 ----- G1的记忆集和卡表更为复杂且Region数量比传统收集器的分代数量要多得多,因此G1收集器要比其他的传统垃圾收集器有着更高的内存占用负担.根据经验,G1至少要耗费大约相当于Java堆容量10%至20%的额外内存来维持收集器工作.
* G1的负载问题 ----- G1的记忆集卡表更复杂,维护时使用写前屏障和写后屏障,而CMS只用到写后屏障,所以G1执行负载更大.但是原始快照方式比增量更新方式停顿更短(因为原始快照搜索能够减少并发标记和重新标记阶段的消耗).
* G1的并发标记的问题 ----- 采用原始快照(SATB)实现.垃圾收集过程与用户线程并发,因此就会有新对象产生分配,G1位每个Region设计了两个名为TAMS(Top at Mark Start)的指针,把Region中的一部分空间划分出来用于并发回收过程中的新对象分配,并发回收时新分配的对象地址都必须要在这两个指针位置以上,G1收集器默认在这个地址以上的对象是被隐式标记过的,即默认它们是存活的,不纳入回收范围.与CMS中"Concurrent Mode Failure"失败会导致Full GC类似,如果内存回收的速度赶不上内存分配的速度,G1收集器也要被迫冻结用户线程执行,导致Full GC而产生长时间STW.


G1收集器运行示意图:
[![6i7QUI.png](https://s3.ax1x.com/2021/03/01/6i7QUI.png)](https://imgtu.com/i/6i7QUI)

G1收集器 Mixed GC的运作过程大致可划分为以下四个步骤:<font color="red">注意,G1只针对年轻代进行young gc时是没有下面步骤的</font>
* 初始标记: 仅仅只是标记一下GC Roots能直接关联到的对象(可理解为根节点枚举),并且修改TAMS指针的值,让下一阶段用户线程并发运行时,能正确地在可用的Region中分配新对象.这个阶段需要停顿线程,但耗时很短,而且是借用进行Minor GC的时候同步完成的,所以G1收集器在这个阶段实际没有额外的停顿.
* 并发标记: 从GC Roots开始对堆中对象进行可达性分析,递归扫描整个堆里的对象图,找出要回收的对象,这阶段耗时较长,但可与用户程序并发执行.当对象图扫描完成以后,还要重新处理SATB(原始快照)记录下的在并发时有引用变动的对象.
* 最终标记: 对用户线程做另一个短暂的暂停,用于处理并发阶段结束后仍遗留下来的最后那少量的SATB记录.
* 筛选回收: 负责更新Region的统计数据,对各个Region的回收价值和成本进行排序,根据用户设定的目标停顿时间来制定回收计划,把决定回收的那一部分Region的存活对象复制到空的Region中,再清理掉整个旧Region的全部空间.这里的操作涉及存活对象的移动,是必须暂停用户线程的,由多条收集器线程并行完成.

因此,G1收集器除了并发标记阶段外,其余阶段都是要完全暂停用户线程的.<br>


G1与CMS的对比:<br>

优点: 
* 可指定最大停顿
* 整体基于"标记-整理",局部基于"标记-复制",因此运作期间不会产生内存碎片,垃圾收集完成后能提供规整的可用内存.这种特性有利于程序长时间运行,不易因大对象分配找不到连续空间而提前触发下一次收集.对堆内存大的类型的服务程序适应更好(部分Region收集,非整代,因此更快,只要收集速度大于分配速度即可).

弱项:
* 内存占用高于CMS
* 额外执行负载高于CMS


<font color="red">G1收集器,在只对年轻代进行Minor GC的时候,是不执行并发标记阶段的,进行Minor GC的过程中,STW.只有在Mixed GC时(年轻代+部分老年代)才进行并发标记阶段.换句话讲,并发标记周期是为Mixed GC服务的,这个阶段将会为混合收集周期识别垃圾最多的老年代分区.并发标记阶段是借用young gc阶段的STW完成初始标记的.</font><br>

当达到如下条件时,会开启并发标记阶段:
* 当一次young gc时,计算和评估后,老年代region占用空间总量对整堆空间总量的百分比大于-XX:InitiatingHeapOccupancyPercent(默认45%),这个参数的解释,网上说什么的都有,有的说是整堆使用占比超过45%,有的说是老年代占用超过整堆45%.经试验验证,是老年代region占用空间总量对整堆空间总量的占比,超过整堆的45%
* 可用空间小于-XX:G1ReservePercent
* 遇到大对象分配的特殊情况

Evacuation Failure,在G1中指:转移失败.可类比于CMS收集器的Concurrent Mode Failure.下面是官方文档对其的解释
[![6oHvB8.png](https://z3.ax1x.com/2021/03/22/6oHvB8.png)](https://imgtu.com/i/6oHvB8)

当出现Evacuation Failure时,会进行Full GC,STW.因此,为了避免Evacuation Failure,G1会预留一部分空间,预留百分比由参数-XX:G1ReservePercent来控制,默认10%.当然这也并不能绝对保证不会出现避免Evacuation Failure.<br>

-XX:G1HeapWastePercent参数的含义:<br>
标记周期完成后,统计出的所有Cset内,老年代Region中可回收的垃圾占比大于G1HeapWastePercent指定的百分比时,才会开启Mixed GC,最多执行G1MixedGCCountTarget次Mixed GC,默认8次,当执行完某次Mixed GC时,比如执行了4次,如果此时老年代Region中可回收的垃圾占比小于G1HeapWastePercent了,就不再继续执行下次Mixed GC了.<br>

-XX:G1MixedGCCountTarget参数的含义:<br>
Mixed GC中,它将用候选Old区域的数量除以G1MixedGCCountTarget,并尝试在每个周期中至少收集那么多区域,最多会收集G1MixedGCCountTarget次.<br>

-XX:G1MixedGCLiveThresholdPercent参数的含义:<br>
Mixed GC中,老年代Region中存活对象低于G1MixedGCLiveThresholdPercent(默认85%)指定的百分比时,Mixed GC时才会对其回收.<br>

在满足触发并发标记的条件后,并不会立即开启并发标记周期,而是等待一次young gc,并复用这次young gc的STW完成的root scan操作,因此可以说全局并发标记总是伴随着young gc而发生的.

<br><br>
## <span id="jump9">九. 低延迟垃圾垃圾收集器</span>

<br>
**<font size="5">Shenandoah 收集器</font>**<br>

OpenJDK12中可以使用,OracleJDK中不支持.但Shenandoah收集器才更像G1的继承者.<br>

其目标是实现一种在任何堆内存大小下都可以把垃圾收集的停顿时间限制在十毫秒以内的垃圾收集器,该目标意味着相比CMS和G1,Shenandoah不仅要进行并发的垃圾标记,还要并发地进行对象清理后的整理动作.<br>

Shenandoah也是基于Region的堆内存布局,同样有着用于存放大对象的Humongous Region,默认回收策略也同样是优先处理回收价值最大的Region.<br>

Shenandoah对G1的改进:
* 支持并发的整理算法 ----- <font color="red">读屏障和"Brooks Pointers(转发指针)"</font>
* 默认不使用分代收集,换言之,不会有专门的新生代Region或者老年代Region存在,没有实现分代并不是说分代对Shenandoah没有价值,更多是处于性价比的权衡,将其放在优先级较低的工作中.
* 摒弃了G1中耗费大量内存和计算资源去维护的记忆集,改用名为"<font color="red">连接矩阵(Connection Matrix)</font>"的全局数据结构来记录跨Region的引用关系.

连接矩阵原理示意图:
[![6FdhJx.png](https://s3.ax1x.com/2021/03/02/6FdhJx.png)](https://imgtu.com/i/6FdhJx)


Shenandoah收集器的工作过程大致可分为以下九个阶段:
* 初始标记: 同G1,首先标记与GC Roots直接关联的对象,可理解为根节点枚举,STW.停顿时间与堆大小无关,至于GC Roots数量有关(得益于Oop Map)
* 并发标记: 同G1,遍历图对象,与用户线程并发,耗时长短取决于堆中存活对象的数量及对象图的结构复杂程度
* 最终标记: 同G1,处理SATB扫描,并在这个阶段统计出回收价值最高的Region,构成回收集(Collection Set,CSet).短暂STW
* 并发清理: 与G1有区别,该阶段用于清理那些整个区域内一个存活对象都没找到的Region
* 并发回收: Shenandoah与之前其他HotSpot收集器的核心差异.把回收集中的存活对象复制一份到其他未被使用的Region之中,与用户线程并发执行.对于与用户线程并发的新旧引用地址问题,通过读屏障和"Brooks Pointers(转发指针)"来解决
* 初始引用更新: 把堆中所有指向旧对象的引用修正到复制后的新地址,这个操作称为"引用更新".该阶段其实没做具体处理,只是建立一个线程集合点,确保所有并发回收阶段中进行的收集器线程都已完成对象移动任务.会产生一个非常短暂的STW
* 并发引用更新: 真正开始进行引用更新操作,与用户线程并发.
* 最终引用更新: 解决完堆中的引用更新之后,还要修正存在于GC Roots中的引用.这个阶段是Shenandoah最后一次STW,停顿时间至于GC Roots的数量有关.
* 并发清理: 经过并发回收和引用更新之后,整个回收集中所有的Region已再无存活对象,最后再调用一次并发清理过程来回收这些Region的内存空间,供以后新对象分配使用

**<font color="red">其中三个重要并发阶段: 并发标记、并发回收、并发引用更新</font>**


Shenandoah在实际应用中的测试数据:(使用ElasticSearch对200G维基百科数据进行索引)
[![6FsRTe.png](https://s3.ax1x.com/2021/03/02/6FsRTe.png)](https://imgtu.com/i/6FsRTe)

停顿时间有了质的飞跃,但未实现最大停顿控制在十毫秒下的目标,而吞吐量方面出现了明显下降.<br>

因此可看出  <font color="red">Shenandoah的弱项:高运行负担使得吞吐量下降;强项:低延迟时间</font>


<br>
**<font size="5">ZGC 收集器</font>**<br>

ZGC和Shenandoah的目标是高度相似的,都希望在尽可能对吞吐量影响不太大的前提下,实现在任意堆内存大小下都可以把垃圾收集的停顿时间限制在十毫秒以内的低延迟.<br>

ZGC收集器是一款基于Region内存布局的,(暂时)不设分代的,使用了读屏障、染色指针和内存多重映射等技术来实现可并发的标记-整理算法的,以低延迟为首要目标的一块垃圾收集器.<br>

内存布局方面,ZGC与Shenandoah和G1一样,也采用Region的堆内存布局,但不同的是,ZGC的Region(一些官方资料中将它称为Page和ZPage)具有动态性 ----- 动态创建和销毁,以及动态的区域容量大小.<br>

X64硬件平台下,<font color="red">ZGC的Region可具有大、中、小三类容量</font>:
* 小型Region: 容量固定2MB,用于防止小于256KB的小对象
* 中型Region: 容量固定32MB,用于防止大于等于256KB,但小于4MB的对象
* 大型Region: 容量不固定可动态变化,但必须为2MB的整数倍,用于放置4MB以上大对象.每个大型Region只会存放一个大对象.大型Region的实际容量完全可能小于中型Region,最小容量可低至4MB.大型Region在ZGC的实现中不会被重新分配,因为复制一个大对象的代价非常高.


<font color="red">并发整理算法实现方面,读屏障 + 染色指针技术 + 转发表(处理引用问题,可指针"自愈")</font> <br>

染色指针直接把标记信息记在引用对象的指针上,使得虚拟机可以直接通过指针就能看到其引用对象的三色状态、是否进入重分配集(即被移动过)等,这时,与其说可达性分析是遍历对象,还不如说是遍历"引用图"来标记引用.<br>

<font color="red">为什么使用染色指针: 染色指针可以大幅减少在垃圾收集过程中内存屏障的使用数量.</font> <br>

ZGC的运作过程大致可划分为四个大的阶段,全部四个阶段都是可以并发执行的,仅是两个阶段中间会存在短暂停顿的小阶段:
[![6kmabF.png](https://s3.ax1x.com/2021/03/02/6kmabF.png)](https://imgtu.com/i/6kmabF)

* 并发标记: 与G1和Shenandoah一样,做可达性分析,前后也要经过类似的初始标记,最终标记(尽管ZGC中的名字不叫这些)的短暂停顿.不同的是ZGC的标记是在指针上而不是在对象上进行的
* 并发预备重分配: 根据特定查询条件统计要清理的Region,组成重分配集(Relocation Set).重分配集与G1收集器的回收集(Collection Set)还是有区别的,ZGC划分Region的目的并非为了像G1那样做收益优先的增量回收.相反,ZGC每次回收都会扫描所有的Region,用范围更大的扫描成本换取省去G1中记忆集的维护成本.因此,ZGC的重分配集只是决定了里面的存活对象会被重新复制到其他的Region,重分配集里面的Region会被释放,而并不能说回收行为就只是针对这个集合里面的Region进行,因为标记过程是针对全堆的
* 并发重分配: 重分配是ZGC执行过程中的核心阶段,这个过程要把重分配集合中的存活对象复制到新的Region上,并为重分配集中的每个Region维护一个转发表,记录从旧对象到新对象的转向关系.得益于染色指针的支持,ZGC收集器能仅从引用上就明确得知一个对象是否处于重分配集中(也即可以理解为对象是否被移动),如果用户线程此时并发访问了位于重分配集中的对象,这次访问将会被预置的内存屏障所截获,然后立即根据Region上的转发表将访问转发到新复制的对象,并同时修正更新该引用的值,使其直接指向新对象,ZGC将这种行为称为指针的"自愈"能力.这样做的好处是只有第一次访问旧对象会陷入转发,也就是只慢一次,对比Shenandoah的Brooks转发指针,那是每次对象访问都必须付出的固定开销,简单地说就是每次都慢,因此ZGC对用户程序的运行时负载要比Shenandoah来的更低一些.还有另外一个直接的好处是由于染色指针的存在,一旦重分配集中某个Region的存活对象都复制完毕后,这个Region就可以立即释放用于新对象的分配(但是转发表还得留着不能释放掉),哪怕堆中还有很多指向这个对象的未更新指针也没有关系,这些旧指针一旦被使用,它们都是可以自愈的.
* 并发重映射: 重新映射所做的就是修正整个堆中指向重分配集中旧对象的所有引用,因为就指针引用能够自愈,所以这一任务不是很迫切,ZGC将其合并到了下一次垃圾收集循环中的并发标记阶段里去完成了,反正它们都是要遍历所有对象的,这样合并就节省了一次遍历对象图的开销.

相比G1,Shenandoah等先进的垃圾收集器,ZGC在实现细节上做了一些不同的权衡选择,譬如G1需要通过写屏障来维护记忆集,才能处理跨代指针,得以实现Region的增量回收.记忆集要占用大量的内存空间,写屏障也对正常程序运行造成额外负担,这些都是权衡选择的代价.ZGC就完全没有使用记忆集,他甚至连分代都没有,所以也不需要记录跨代间引用的卡表,因此完全没有用到写屏障,所以给用户线程带来的运行负担小得多.但同时,这种取舍也限制了它能承受的对象分配速率不会太高,因为他的扫描范围会大,因此慢,但能并发,造成的用户STW短.(这里务必区分开并发时间与停顿时间,ZGC的目标是停顿时间不超过十毫秒,但是整个收集周期的时间是比较长的)
