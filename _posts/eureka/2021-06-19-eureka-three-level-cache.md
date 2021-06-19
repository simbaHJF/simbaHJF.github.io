---
layout:     post
title:      "Eureka"
date:       2021-06-19 00:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - eureka

---











[![RC6cVI.png](https://z3.ax1x.com/2021/06/19/RC6cVI.png)](https://imgtu.com/i/RC6cVI)



[![RP3D9P.png](https://z3.ax1x.com/2021/06/19/RP3D9P.png)](https://imgtu.com/i/RP3D9P)





<br><br>
## <span id="jump2">二. 为什么这样设计</span>

eureka所设计的缓存级别无疑也是为了读写分离,因为在写的时候,如ConcurrentHashmap会持有桶节点对象的锁,阻塞同一个桶的读写线程.这样设计的话,线程在写的时候,并不会影响读操作,避免了争抢资源所带来的压力.<br>



