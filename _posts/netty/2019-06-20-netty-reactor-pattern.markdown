---
layout:     post
title:      "reactor线程模型和netty线程模型"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

>参考资料:<br>
 netty权威指南


写在最前面的话,Reactor到底是什么呢?说白了就是一个运行中的线程.

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


这里以 JAVA NIO 做样例,描述一下 Reactor 单线程模型.
```
public class ReactorSingle implements Runnable{

    private Selector selector;
    private ServerSocketChannel serverSocketChannel;
    private static final int port = 8086;

    public ReactorSingle() {
        try {
            selector = Selector.open();
            serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().bind(new InetSocketAddress(port),1024);
            serverSocketChannel.register(selector, SelectionKey.OP_CONNECT);
        } catch (Exception e) {

        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                SelectionKey key = null;
                while (iterator.hasNext()) {
                    key = iterator.next();
                    iterator.remove();
                    handle(key);
                }
            } catch (Exception e) {

            }
        }
    }

    private void handle(SelectionKey key) throws IOException {
        if (key.isValid()) {
            if (key.isAcceptable()) {
                ServerSocketChannel serverSocketChannel = (ServerSocketChannel)key.channel();
                SocketChannel sc = serverSocketChannel.accept();
                sc.configureBlocking(false);
                sc.register(this.selector, SelectionKey.OP_READ);
            }
            if (key.isReadable()) {
                //这里假设数据不会超过1024个字节长度
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                sc.read(readBuffer);
                readBuffer.flip();
                byte[] bytes = new byte[readBuffer.remaining()];
                readBuffer.get(bytes);

                // ....对bytes的一些业务处理,省略

                String resp = "OK";
                byte[] respBytes = resp.getBytes();
                ByteBuffer writeBuffer = ByteBuffer.allocate(respBytes.length);
                writeBuffer.put(respBytes);
                writeBuffer.flip();
                sc.write(writeBuffer);
            }
            // ....省略一些其他
        }
    }

    public static void main(String[] args) {
        ReactorSingle reactorSingle = new ReactorSingle();
        new Thread(reactorSingle).start();
    }
}
```

Reactor单线程模型,总结起来就是,连接,读写,业务处理,全都自己干且在自己这个本线程内干.示例图中的handler包含着读写以及业务处理逻辑,就如同上面描述代码中的handle函数一样,只不过上述代码中,为了简单,以方法形式写了,没有将其抽象出来最为一个handler对象而已.对应读写,就是调用handler对象处理,对于连接就是调用acceptor对象处理.


##	Reactor多线程模型
Reactor多线程模型与单线程模型最大的区别是有一组NIO线程来处理I/O操作,它的原理如图所示:
[![](https://s2.ax1x.com/2019/06/20/VxJobF.md.png)](https://imgchr.com/i/VxJobF)
Reactor多线程模型的特点如下:
*	有专门一个NIO线程----Acceptor线程用于监听服务端,接收客户端的TCP连接请求.
*	网络I/O操作----读,写等操作由一个NIO线程池负责,线程池可以采用标准的JDK线程池实现,它包含一个任务队列和N个可用的线程,由这些NIO线程负责消息的读取,解码,编码和发送.
*	一个NIO线程可以同时处理N条链路,但是一个链路只对应一个NIO线程,防止发生并发操作问题.

在绝大多数场景下,Reactor多线程模型可以满足性能需求.但是,在个别特殊场景中,一个NIO线程负责监听和处理所有的客户连接可能会存在性能问题.例如并发百万客户端连接,或者服务端需要对客户端握手进行安全认证,但是认证本身非常损耗性能.在这类场景下,单独一个Acceptor线程可能会存在性能不足的问题,为了解决性能问题,产生了第三种Reactor线程模型----主从Reactor多线程模型<br>


拿前面的JAVA NIO 描述代码来讲, Reactor多线程模型与Reactor单线程模型的区别就在于:<br>
Reactor多线程模型中,Reactor这个线程内只做这几件事:
* 发现连接就绪事件,在Reactor本线程内,调用acceptor完成连接的accept处理和注册
* 发现读写就绪事件,封装成一个任务,任务中调用相应的handler完成读写和业务处理工作.而这个任务是丢到另外的一个线程池中去做的,不在Reactor本线程中搞了.也就是说,对于读写就绪事件,还是由Reactor线程来发现的,但后面具体的读写和业务处理,就交给小弟(线程池)去弄了.





##	主从Reactor多线程模型
主从Reactor线程模型的特点是:服务端用于接收客户端连接的不再是一个单独的NIO线程,而是一个独立的NIO线程池.Acceptor接收到客户端TCP连接请求并处理完成后(可能包含接入认证等),将新创建的SocketChannel注册到I/O线程池(sub Reactor线程池)的某个I/O线程上,由它负责SocketChannel的读写和编码工作.Acceptor线程池仅仅用于客户端的登录,握手和安全认证,一旦链路建立成功,就将链路注册到后端sub Reactor线程池的I/O线程上,由I/O线程负责后续的I/O操作.它的线程模型如下图所示:
[![](https://s2.ax1x.com/2019/06/21/VxtIk4.md.png)](https://imgchr.com/i/VxtIk4)
利用主从NIO线程模型,可以解决一个服务端监听线程无法有效处理所有客户端连接的性能不足问题.Netty的官方推荐采用主从Reactor线程模型.

<br><br><br>

**<font color="red">总结一下,Reactor多线程模型和主从Reactor多线程模型的差别</font>**
*	Reactor多线程模型中,Reactor既负责连接事件,也负责读写事件(只是获取相关的事件,具体的读写处理,交由NIO读写线程池处理)
*	主从Reactor多线程模型中,mainReactor只负责连接事件以及连接处理的这块操作,之后就将连接交由subReactor(可能是一个线程,也可能是一组线程)负责读写事件,而subReactor也采用甩锅这一套,只负责发现读写事件的就绪,将具体的读写和业务处理封装成任务(任务中调用相应handler),丢给线程池处理.




##	Netty的线程模型
通过设置不同的参数,Netty可以支持Reactor单线程模型,多线程模型和主从Reactor多线程模型.Netty线程模型的原理图如下:
![](https://s2.ax1x.com/2019/07/04/ZaCin0.png)

服务端启动的时候,netty通常会创建两个EventLoopGroup,也就是主从Reactor线程模型.

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

第一个被称为boss线程组,用于接收连接请求;第二个称为worker线程组,用来处理已连入的连接在其生命周期内所发生的事件(I/O读写操作等).<br>


##	netty最佳实践
1.创建两个NioEventLoopGroup,用于逻辑隔离NIO Acceptor和NIO I/O线程.<br>
2.尽量不要在ChannelHandler中启动用户线程(解码后用于将POJO消息派发到后端业务线程的除外).<br>
3.解码要放在NIO线程调用的解码Handler中进行,不要切换到用户线程中完成消息的解码.<br>
4.如果业务逻辑操作非常简单,没有复杂的业务逻辑计算,没有可能会导致线程被阻塞的磁盘操作,数据库操作,网络操作等,可以直接在NIO线程上完成业务逻辑编排,不需要切换到用户线程.<br>
5.如果业务逻辑处理复杂,不要在NIO线程上完成,建议将解码后的POJO消息封装成Task,派发到业务线程池中由业务线程执行,以保证NIO线程尽快被释放,处理其他的I/O操作.<br>
