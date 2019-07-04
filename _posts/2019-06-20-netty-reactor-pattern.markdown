---
layout:     post
title:      "reactor线程模型"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

>参考资料:<br>
 netty权威指南


##	Reactor单线程模型
Reactor单线程模型,是指所有的I/O操作都在一个NIO线程上完成,NIO线程完成如下功能:
*	作为NIO服务端,接收客户端的TCP连接
*	作为NIO客户端,向服务端发起TCP连接
*	读取通信对端的请求或应答消息
*	向通信对端发送消息请求或者应答消息

Reactor单线程模型如下图:
[![](https://s2.ax1x.com/2019/06/20/VxGkod.md.png)](https://imgchr.com/i/VxGkod)
Reactor单线程模型中,Reactor,Acceptor和Handler在同一个线程中,多路复用器阻塞轮询事件,有事件触发后,调用相应的Handler处理.<br>
在一些小容量应用场景下,可以使用单线程模型.但是对于高负载,大并发的应用场景不合适,原因如下:
*	一个NIO线程同时处理成百上千的链路,性能上无法支撑,即便NIO线程的CPU负荷达到100%,也无法满足海量消息的编码,解码,读取和发送.
*	当NIO线程负载过重之后,处理速度将变慢,这会导致大量客户端连接超时,超时之后往往会进行重发,这更加重了NIO线程的负载,最终会导致大量消息积压和处理超时,成为系统的性能瓶颈.
*	可靠性问题:一旦NIO线程意外跑飞,或者进入死循环,会导致整个系统通信模块不可用,不能接受和处理外部消息,造成节点故障.


##	Reactor多线程模型
Reactor多线程模型与单线程模型最大的区别是有一组NIO线程来处理I/O操作,它的原理如图所示:
[![](https://s2.ax1x.com/2019/06/20/VxJobF.md.png)](https://imgchr.com/i/VxJobF)
Reactor多线程模型的特点如下:
*	有专门一个NIO线程----Acceptor线程用于监听服务端,接收客户端的TCP连接请求.
*	网络I/O操作----读,写等操作由一个NIO线程池负责,线程池可以采用标准的JDK线程池实现,它包含一个任务队列和N个可用的线程,由这些NIO线程负责消息的读取,解码,编码和发送.
*	一个NIO线程可以同时处理N条链路,但是一个链路只对应一个NIO线程,防止发生并发操作问题.

在绝大多数场景下,Reactor多线程模型可以满足性能需求.但是,在个别特殊场景中,一个NIO线程负责监听和处理所有的客户连接可能会存在性能问题.例如并发百万客户端连接,或者服务端需要对客户端握手进行安全认证,但是认证本身非常损耗性能.在这类场景下,单独一个Acceptor线程可能会存在性能不足的问题,为了解决性能问题,产生了第三种Reactor线程模型----主从Reactor多线程模型


##	主从Reactor多线程模型
主从Reactor线程模型的特点是:服务端用于接收客户端连接的不再是一个单独的NIO线程,而是一个独立的NIO线程池.Acceptor接收到客户端TCP连接请求并处理完成后(可能包含接入认证等),将新创建的SocketChannel注册到I/O线程池(sub Reactor线程池)的某个I/O线程上,由它负责SocketChannel的读写和编码工作.Acceptor线程池仅仅用于客户端的登录,握手和安全认证,一旦链路建立成功,就将链路注册到后端sub Reactor线程池的I/O线程上,由I/O线程负责后续的I/O操作.它的线程模型如下图所示:
[![](https://s2.ax1x.com/2019/06/21/VxtIk4.md.png)](https://imgchr.com/i/VxtIk4)
利用主从NIO线程模型,可以解决一个服务端监听线程无法有效处理所有客户端连接的性能不足问题.Netty的官方推荐采用主从Reactor线程模型.


##	Netty的线程模型
通过设置不同的参数,Netty可以支持Reactor单线程模型,多线程模型和主从Reactor多线程模型.Netty线程模型的原理图如下:
![](https://s2.ax1x.com/2019/07/04/ZaCin0.png)

服务端启动的时候,netty通常会创建两个EventLoopGroup,也就是主从Reactor线程模型.

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

第一个被称为boss线程组,用于接收连接请求;第二个称为worker线程组,用来处理已连入的连接在其生命周期内所发生的事件(I/O读写操作等).