---
layout:     post
title:      "JVM优化方法论"
date:       2021-03-04 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JVM

---







<br><br>
## <span id="jump1">一. 一个新系统开发完毕之后如何设置JVM参数</span>

首先应该预估一下系统的每个核心接口每秒有多少次请求,每次请求会创建多少对象,每个对象大约占多大空间,以此来估算每秒会使用多少内存空间.<br>

接下来,就可以估算出Eden区大概多长时间会占满.<br>

然后,就能以此估算出多长时间会触发以此Young GC,以及发生Young GC时,会有多少对象存活下来,有多少对象升入老年代.<br>

最后,以此就可以估算老年代的增长速率大概是多少,多久会触发一次Full GC.<br>

通过这一连串的估算,就可以大约的社指出合理的年轻代老年代空间大小,还有Eden和Survior空间大小.<br>

原则就是: 尽可能让每次Young GC后存活的对象远远小于Survior区域,避免对象拼房进入老年代触发Full GC.最理想的状态下,就是系统几乎不发生Full GC,老年代应该就是稳定占用一定的空间,就是那些长期存活的对象在躲过15次Young GC后升入老年代自然占用的.然后平时主要就是几分钟发生一次Young GC,耗时几毫秒.<br>



<br><br>
## <span id="jump2">二. 在压测之后合理调整JVM参数</span>

通过压测模拟线上压力的场景下,使用jstat等工具去观察JVM的运行情况:
* Eden区的对象增长速率多快
* Young GC频率多高
* 一次Young GC多长耗时
* Young GC过后多少对象存活
* 老年代的对象增长速率多高
* Full GC频率多高
* 一次Full GC耗时

压测时,可以通过jstat精准的观察出上述JVM运行指标,然后对JVM参数做相关调整,避免对象频繁进入老年代,尽量让系统仅有Young GC.<br>



<br><br>
## <span id="jump3">三. 线上系统的监控和优化</span>

系统上线后,进行相关的JVM监控,频繁Full GC要进行报警.<br>



<br><br>
## <span id="jump4">四. 线上频繁Full GC的几种表现</span>

一旦系统发生频繁Full GC,大概能看到的一些表象如下:
* 机器CPU负载过高
* 频繁Full GC报警
* 系统无法处理请求或者处理过慢

所以一旦发生上述几个情况,第一时间可以考虑排查是否发生了频繁Full GC.<br>



<br><br>
## <span id="jump5">五. 频繁Full GC的几种常见原因</span>

1. 系统承载高并发请求,或者处理数据量过大,导致Young GC很频繁,而且每次Young GC过后存活对象太多,内存分配不合理,Survior区域过小,导致对象频繁进入老年代,频繁触发Full GC
2. 系统一次性加载过多数据进内存,搞出很多大对象,导致频繁有大对象进入老年代,造成频繁Full GC
3. 系统发生内存泄露,莫名创建大量大对象,始终无法回收,一直占用在老年代里,造成频繁Full GC
4. Metaspace因为加载类过多而Full GC
5. 误调用System.gc()触发Full GC

如果jstat分析发现Full GC原因是第一种,那么就合理分配内存,调大Survior区域即可.<br>

如果jstat分析发现是第二种或第三种原因,也就是老年代一直有大量对象无法回收,年轻代升入老年代的对象并不多,那就dump出来内存快照,然后用MAT工具进行分析即可.通过分析,找出来什么对象占用内存过多,然后通过一些对象的引用和线程执行堆栈的分析,找到哪块代码弄出来的那么多的对象,接着优化代码即可.<br>
``
jmap -dump:live,format=b,file=文件名 [服务进程ID]
``


<br>
**<font size="4">注意!!!!</font>** <br>

<font color="red">jmap不要线上直接使用,说通过jmap分析,那只是口头上的理论可行方案,jmap比较耗时,而且为了进行堆一致性快照信息dump,是会暂停进程的运行的.线上服务直接jmap,是不想过了吗???,线上服务,可以考虑先摘掉服务,然后再通过jmap来dump,或者使用gcore命令来进行dump,比jmap快很多,然后再通过工具分析.</font>
