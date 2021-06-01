---
layout:     post
title:      "kafka 时间轮"
date:       2021-06-01 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. kafka时间轮结构](#jump1)
<br>
[二. kafka时间轮使用场景](#jump2)
<br>






<br><br>
## <span id="jump1">一. kafka时间轮结构</span>

kafka采用时间轮来进行延时任务操作,为什么不适用原生JDK的DelayQueue呢? <br>

因为DelayQueue底层数据结构是一个小顶堆,当存在大量定时任务时,入堆出堆维护就需要维护堆结构,时间复杂度为O(nlogn),无法满足高性能要求.<br>

kafka时间轮采用多级时间轮方式,而不是dubbo中那种 '単级时间轮 + epoch' 的方式.其结构大体如下:

[![2u0OBR.png](https://z3.ax1x.com/2021/06/01/2u0OBR.png)](https://imgtu.com/i/2u0OBR)

[![2uBMvQ.png](https://z3.ax1x.com/2021/06/01/2uBMvQ.png)](https://imgtu.com/i/2uBMvQ)



<br><br>
## <span id="jump2">二. kafka时间轮使用场景</span>

这里只举例两个场景例子,实际应用不止这两种:
* producer生产消息,设置acks=-1,即要求所有ISR集合中的副本都已经复制了消息,leader才会给producer返回响应,同时producer设置了请求的超时时间.那么此时,broker就会通过创建一个延时任务来检测超时,超时则立即向producer客户端返回超时结果.这块和dubbo超时检测很像
* consumer延时拉取,没有最新消息时,现在broker处阻塞一段,这里也是通过添加一个延时任务,等待一段时间再返回的.



