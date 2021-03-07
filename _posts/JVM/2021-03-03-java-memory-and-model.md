---
layout:     post
title:      "JAVA内存模型与线程"
date:       2021-03-03 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JVM

---



## 导航
[一. 处理器、高速缓、主内存间的交互关系](#jump1)
<br>
[二. 线程、主内存、工作内存间的交互关系](#jump2)
<br>
[三. volatile](#jump3)
<br>
[四. 线程状态](#jump4)
<br>






<br><br>
## <span id="jump1">一. 处理器、高速缓、主内存间的交互关系</span>

[![6Epz2F.png](https://s3.ax1x.com/2021/03/03/6Epz2F.png)](https://imgtu.com/i/6Epz2F)



<br><br>
## <span id="jump2">二. 线程、主内存、工作内存间的交互关系</span>

[![6E9txg.png](https://s3.ax1x.com/2021/03/03/6E9txg.png)](https://imgtu.com/i/6E9txg)



<br><br>
## <span id="jump3">三. volatile</span>

**<font size="4">可见性</font>**<br>

保证volatile变量对所有线程的可见性,这里的"可见性是指"当一条线程修改了这个变量的值,新值对于其他线程来时是可以立即得知的.<br>


**<font size="4">禁止指令重排序</font>**<br>

内存屏障,确保不能把后面的指令重排序到内存屏障之前的位置,同时将本处理的缓存写入内存,这就意味着所有之前的操作都已经执行完,这样便形成了"指令重排序无法越过内存屏障"的效果,该写入动作也会引起别的处理器或者别的内核无效化其缓存.<br>



<br><br>
## <span id="jump4">四. 线程状态</span>

Java语言定义了6中线程状态,在任意一个时间点中,一个线程只能有且只有其中的一种状态,并且可以通过特定的方法在不同状态之间转换.
* 新建(New): 创建后尚未启动的线程处于这种状态
* 运行(Runnable): 包括正在运行和就绪(等待操作系统分配执行时间)
* 无限期等待(Waiting): 处于这种状态的线程不会被分配处理器执行时间,它们要等待被其他线程显式唤醒
* 限期等待(Timed Waiting): 处于这种状态的线程也不会被分配处理器执行时间,不过无须等待被其他线程显式唤醒,在一定时间之后它们会由系统自动唤醒.
* 阻塞(Blocked)
* 结束(Terminated): 已终止线程的线程状态,线程已经结束执行

[![6Egty9.png](https://s3.ax1x.com/2021/03/03/6Egty9.png)](https://imgtu.com/i/6Egty9)