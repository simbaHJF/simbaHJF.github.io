---
layout:     post
title:      "netty业务处理"
date:       2019-11-23 22:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

>  极客时间<br>
   netty源码剖析与实战


#	主线

![Mq2uE4.png](https://s2.ax1x.com/2019/11/23/Mq2uE4.png)


*	处理业务本质:数据在pipeline中所有的handler的channelRead执行过程<br>
Handler要实现io.netty.channel.ChannelInboundHandler#channelRead(ChannelHandlerContext ctx, Object msg),且不能加注解@Skip才能被执行到.<br>
中途可退出,不保证执行到TailHandler.

*	默认处理线程就是Channel绑定的NioEventLoop线程,也可以设置其他:<br>
pipeline.addLast(new UnorderedThreadPoolEventExecutor(10), serverHandler).同时,在这里执行pipeline.addLast的时候,会算出一个executionMask,该executionMask用来判断是否具有某个事件(读?写?还是连接等等)的执行资格.


