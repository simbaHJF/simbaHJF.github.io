---
layout:     post
title:      "netty中的组件和设计"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

##	Channel接口
JDK中的Channel----NIO类库中的一个重要组成部分.java.nio.SocketChannel和java.nio.ServerSocketChannel,用于网络通信中非阻塞的I/O操作.<br>

类似于NIO的Channel,Netty提供了自己的Channel和其子类实现,用于异步I/O操作和其他相关操作.io.netty.channel.Channel是Netty网络操作抽象类,它聚合了一组功能,包括但不限于网络的读,写,连接等操作;也包含了Netty框架相关的一些功能,包括获取该Channel的EventLoop,获取缓冲分配器ByteBufAllocator和pipeline等.<br>

Channel是Netty抽象出来的网络I/O读写相关的接口,不使用JDK原生Channel而另起炉灶的原因如下:
*	JDK的SocketChannel和ServerSocketChannel没有统一的Channel接口供业务开发者使用,对于用户而言,没有统一的操作师徒,使用起来不方便.
*	JDK的SocketChannel和ServerSocketChannel的主要职责就是网络I/O操作.它们是SPI接口,由具体的虚拟机厂家提供,所以通过集成SPI功能类来扩展功能的难度很大;直接实现ServerSocketChannel和SocketChannel抽象类,其工作量和重新开发一个新的Channel功能类差不多.
*	Netty的Channel需要能够跟Netty的整体架构融合在一起.
*	自定义Channel功能实现更加灵活.
<br>

基于上述原因,Netty重新设计了Channel接口,并且给与了很多不同实现.其主要设计理念如下:
*	在Channel接口层,采用Facade模式进行统一封装,将网络I/O操作,网络I/O相关联的其他操作封装起来,统一对外提供.
*	Channel接口的定义尽量大而全,为io.netty.channel.socket.SocketChannel和io.netty.channel.socket.ServerSocketChannel提供统一的视图,由不同子类实现不同功能,公共功能在抽象父类中实现,最大程度地实现功能和接口的重用.
*	具体实现采用聚合而非包含的方式,将相关功能类聚合在Channel中,由Channel统一负责分配和调度

下面是io.netty.channel.socket.SocketChannel和io.netty.channel.socket.ServerSocketChannel的继承关系图:
![](https://s2.ax1x.com/2019/07/05/ZaZcX4.png)