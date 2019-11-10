---
layout:     post
title:      "netty中的组件和设计"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

> 参考资料:<br>
netty权威指南<br>
netty in action

##	Channel
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

下面是io.netty.channel.socket.nio包下NioSocketChannel和NioServerSocketChannel的继承关系图:
![](https://s2.ax1x.com/2019/07/05/ZdOgjf.png)

![](https://s2.ax1x.com/2019/07/05/ZdOWDS.png)


##	ChannelPipeline和ChannelHandler
Netty将Channel的数据管道抽象为ChannelPipiline,消息在ChannelPipeline中流动和传递.ChannelPipeline持有I/O事件拦截器ChannelHandler的链表,由ChannelHandler对I/O事件进行拦截和处理,可以方便地通过新增和删除ChannelHandler来实现不同的业务逻辑定制,不需要对已有的ChannelHandler进行修改,能够实现对修改封闭和对扩展的支持.<br>

ChannelPipeline是ChannelHandler的容器,它负责ChannelHandler的管理和事件拦截与调度.<br>

一个消息被ChannelPipeline的ChannelHandler链拦截和处理的全过程如下图:
![](https://s2.ax1x.com/2019/07/05/ZdOnXT.png)

消息的读取和发送处理全流程描述如下:<br>

1.底层的SocketChannel read()方法读取ByteBuf,触发ChannelRead事件,由I/O线程NioEventLoop调用ChannelPipeline的fireChannelRead(Object msg)方法,将消息(ByteBuf)传输到ChannelPipeline中.<br>

2.消息一次被HeadHandler,ChannelHandler1,ChannelHandler2......TailHandler拦截和处理,在这个过程中,任何ChannelHandler都可以中断当前的流程,结束消息的传递.<br>

3.调用ChannelHandlerContext的write方法发送消息,消息从TailHandler开始,途经ChannelHandlerN......ChannelHandler2,ChannelHandler1,HeadHandler,最终被添加到消息发送缓冲区等待刷新和发送,在此过程中也可以中断消息的传递,例如当编码失败时,就需要中断流程,构造异常的Future返回.<br>


Netty中的事件分为inbound事件和outbound事件.inbound事件通常由I/O线程触发,例如TCP链路建立事件,链路关闭事件,读事件,异常通知事件等,它对应上图的左半部分.<br>

Outbound事件通常是由用户主动发起的网络I/O操作,例如用户发起的连接操作,绑定操作,消息发送等操作,它对应上图的右半部分.<br>

上述过程可以简单的用下图描绘:
![](https://s2.ax1x.com/2019/07/05/ZdXDqU.png)

入站和出站ChannelHandler可以被安装到同一个CHannelPipeline中,如果一个消息或者任何其他的入站事件被读取,name它会从ChannelPipeline的头部开始流动,冰杯传递给第一个ChannelInboundHandler.这个ChannelHandler不一定会实际地修改数据,具体取决于具体功能,在这之后,数据将会被传递给链中的下一个ChannelInboundHandler.最终,数据将会到达ChannelPipeline的尾端,届时,所有处理就都结束了.<br>

数据的出站运动(即正在被写的数据)在概念上也是一样的.在这种情况下,数据将从ChannelOutboundHandler链的尾端开始流动,知道它到达链的头部为止.在这之后,出站数据将会到达网络传输层,这里显示为Socket.通常情况下,这将触发一个写操作.<br>

可以看到,ChannelPipeline的设计是责任链模式(chain模式)的一种变形.


从应用程序开发人员的角度来看,Netty的主要组件是ChannelHandler,它充当了所有处理入站和出站数据的应用程序逻辑的容器.ChannelHandler的方法是由网络事件触发的,事实上,ChannelHandler可专门用于几乎任何类型的动作,例如将数据从一种格式转化为另外一种格式,或者处理转换过程中所抛出的异常.ChannelPipeline通过ChannelHandler接口来实现事件的拦截和处理,由于channelHandler中的事件种类繁多,不同的ChannelHandler可能只需要关心其中的某一个或者几个事件,所以,通常ChannelHandler只需要继承ChannelHandlerAdapter类然后覆盖自己关心的方法即可.<br>

例如,下面的例子展示了拦截ChannelActive事件,打印TCP链路建立成功日志,代码如下
```
public class MyInboundHandler extends ChannelHandlerAdapter{
	@override
	public void channelActive(ChannelHandlerContext ctx){
		System.out.println("TCP cennected!");
		ctx.fireChannelActive();
	}
}
```

ChannelHandler的接口方法如下:
![](https://s2.ax1x.com/2019/07/05/Zdx2Gj.png)

ChannelHandlerAdapter的继承关系图:
![](https://s2.ax1x.com/2019/07/05/ZdxhMq.png)

可以看到ChannelHandlerAdapter是一种典型的Adapter设计模式<br>


##	EventLoop
运行任务来处理在连接的生命周期内发生的事件是任何网络框架的基本功能.与之相应的编程上的构造通常被称为事件循环----一个Netty使用了interface io.netty.channel.EventLoop来适配的术语.<br>

事件循环的基本思想,可以用下面代码来简单表示:
```
while(!terminated){
	List<Runnable> readyEvents = blockUntilEventsReady();	//阻塞,直到有事件已经就绪可被运行
	for(Runnable ev: readyEvents){
		ev.run();	//循环遍历,并处理所有的事件
	}
}
```

Netty的EventLoop是协同设计的一部分,它采用了两个基本的API:并发和网络编程.首先,io.netty.util.concurrent包构建在JDK的java.util.concurrent包上,用来提供线程执行器.其次,io.netty.channel包中的类,为了与Channel的事件进行交互,扩展了这些接口/类.下图展示了相应的类层次结构:
![](https://s2.ax1x.com/2019/07/06/Z0p6N8.png)

在这个模型中,一个EventLoop将由一个永远都不会改变的Thread驱动,同时任务(Runnable或者Callable)可以直接提交给EventLoop实现,以立即执行或者调度执行,可以理解为:它内部有一个 Thread thread 属性, 存储了一个本地 Java 线程,一个 NioEventLoop 其实和一个特定的线程绑定, 并且在其生命周期内, 绑定的线程都不会再改变.根据配置和可用核心的不同,可能会创建多个EventLoop实例用于优化资源的使用,并且单个EventLoop可能会被指派用于服务多个Channel.<br>

作为NIO框架的Reactor线程,NioEventLoop需要处理网络I/O读写事件,因此它必须内部聚合一个多路复用器对象.可以理解为:每个EventLoop内部维护着一个Selector实例.前面的事件循环的基本思想在NioEventLoop内部,就是通过这个Selector来实现的.


##	Future


##	Promise