---
layout:     post
title:      "Dubbo Remoting Transporter"
date:       2021-01-26 22:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---


## 导航
[一. AbstractPeer抽象类 和 AbstractEndpoint抽象类](#jump1)
<br>
[二. Server 继承路线分析](#jump2)
<br>
[三. Client 继承路线分析](#jump3)




<br><br>
## <span id="jump1">一. AbstractPeer抽象类 和 AbstractEndpoint抽象类</span>

<br>
**<font size="3">AbstractPeer抽象类</font>** <br>

首先,我们来看 AbstractPeer 这个抽象类,它同时实现了 Endpoint 接口和 ChannelHandler 接口,如下图所示,它也是 AbstractChannel、AbstractEndpoint 抽象类的父类
[![svlyxf.png](https://s3.ax1x.com/2021/01/26/svlyxf.png)](https://imgchr.com/i/svlyxf)

AbstractPeer 中有四个字段:一个是表示该端点自身的 URL 类型的字段,还有两个 Boolean 类型的字段(closing 和 closed)用来记录当前端点的状态,这三个字段都与 Endpoint 接口相关;第四个字段指向了一个 ChannelHandler 对象,AbstractPeer 对 ChannelHandler 接口的所有实现,都是委托给了这个 ChannelHandler 对象.从上面的继承关系图中,我们可以得出这样一个结论:AbstractChannel、AbstractServer、AbstractClient 都是要关联一个 ChannelHandler 对象的.<br>

<font color="red">讲道理我个人觉得端点实现ChannelHandler这种接口关系设计,比较不符合逻辑归属关系.小喷一嘴.<br></font>


<br>
**<font size="3">AbstractEndpoint抽象类</font>** <br>

我们顺着上图的继承关系继续向下看,AbstractEndpoint 继承了 AbstractPeer 这个抽象类.AbstractEndpoint 中维护了一个 Codec2 对象(codec 字段)和两个超时时间(timeout 字段和 connectTimeout 字段),在 AbstractEndpoint 的构造方法中会根据传入的 URL 初始化这三个字段
```
public AbstractEndpoint(URL url, ChannelHandler handler) {

    super(url, handler); // 调用父类AbstractPeer的构造方法

    // 根据URL中的codec参数值，确定此处具体的Codec2实现类

    this.codec = getChannelCodec(url);

    // 根据URL中的timeout参数确定timeout字段的值，默认1000

    this.timeout = url.getPositiveParameter(TIMEOUT_KEY,DEFAULT_TIMEOUT);

    // 根据URL中的connect.timeout参数确定connectTimeout字段的值，默认3000

    this.connectTimeout = url.getPositiveParameter(Constants.CONNECT_TIMEOUT_KEY, Constants.DEFAULT_CONNECT_TIMEOUT);

}

```

Codec2 接口是一个 SPI 扩展点,这里的 AbstractEndpoint.getChannelCodec() 方法就是基于 Dubbo SPI 选择其扩展实现的,具体实现如下:
```
protected static Codec2 getChannelCodec(URL url) {
    String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");
    if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) {
        return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
    } else {
        return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class)
                .getExtension(codecName));
    }
}
```

另外,AbstractEndpoint 还实现了 Resetable 接口(只有一个 reset() 方法需要实现),逻辑比较简单,就是根据传入的 URL 参数重置 AbstractEndpoint 的三个字段.


<br><br>
## <span id="jump2">二. Server 继承路线分析</span>

<br>
**<font size="3">AbstractServer 和 AbstractClient</font>** <br>

AbstractServer 和 AbstractClient 都实现了 AbstractEndpoint 抽象类,我们先来看 AbstractServer 的实现.AbstractServer 在继承了 AbstractEndpoint 的同时,还实现了 RemotingServer 接口,如下图所示:
[![svtvn0.png](https://s3.ax1x.com/2021/01/26/svtvn0.png)](https://imgchr.com/i/svtvn0)


AbstractServer 是对服务端的抽象,实现了服务端的公共逻辑.AbstractServer 的核心字段有下面几个
* localAddress、bindAddress(InetSocketAddress 类型):分别对应该 Server 的本地地址和绑定的地址,都是从 URL 中的参数中获取.bindAddress 默认值与 localAddress 一致
* accepts(int 类型):该 Server 能接收的最大连接数,从 URL 的 accepts 参数中获取,默认值为 0,表示没有限制
* executorRepository(ExecutorRepository 类型):负责管理线程池,后面我们会深入介绍 ExecutorRepository 的具体实现
* executor(ExecutorService 类型):当前 Server 关联的线程池,由上面的 ExecutorRepository 创建并管理

在 AbstractServer 的构造方法中会根据传入的 URL初始化上述字段,并调用 doOpen() 这个抽象方法完成该 Server 的启动,具体实现如下
```
public AbstractServer(URL url, ChannelHandler handler) {

    super(url, handler); // 调用父类的构造方法

    // 根据传入的URL初始化localAddress和bindAddress

    localAddress = getUrl().toInetSocketAddress();

    String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());

    int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());

    if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {

        bindIp = ANYHOST_VALUE;

    }

    bindAddress = new InetSocketAddress(bindIp, bindPort);

    // 初始化accepts等字段

    this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);

    this.idleTimeout = url.getParameter(IDLE_TIMEOUT_KEY, DEFAULT_IDLE_TIMEOUT);

    try {

        doOpen(); // 调用doOpen()这个抽象方法,启动该Server,该方法由子类实现.

    } catch (Throwable t) {

        throw new RemotingException("...");

    }

    // 获取该Server关联的线程池

    executor = executorRepository.createExecutorIfAbsent(url);

}
```

ExecutorRepository 负责创建并管理 Dubbo 中的线程池,该接口虽然是个 SPI 扩展点,但是只有一个默认实现—— DefaultExecutorRepository.在该默认实现中维护了一个 ConcurrentMap<String, ConcurrentMap<Integer, ExecutorService>\> 集合(data 字段),用于缓存已有的线程池,第一层 Key 值表示线程池属于 Provider 端还是 Consumer 端,第二层 Key 值表示线程池关联服务的端口.<br>

下面来看基于 Netty 4 实现的 NettyServer,它继承了前文介绍的 AbstractServer,实现了 doOpen() 方法和 doClose() 方法.这里重点看 doOpen() 方法,如下所示:
```
protected void doOpen() throws Throwable {

    // 创建ServerBootstrap
    bootstrap = new ServerBootstrap(); 
    // 创建boss EventLoopGroup
    bossGroup = NettyEventLoopFactory.eventLoopGroup(1, "NettyServerBoss");
    // 创建worker EventLoopGroup
    workerGroup = NettyEventLoopFactory.eventLoopGroup(
            getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
            "NettyServerWorker");
    // 创建NettyServerHandler，它是一个dubbo-remoting-netty4包下,对Netty中的ChannelHandler的一个实现，
    // 不是Dubbo Remoting层的ChannelHandler接口的实现
    final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
    // 获取当前NettyServer创建的所有Channel，这里的channels集合中的
    // Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
    channels = nettyServerHandler.getChannels();
    // 初始化ServerBootstrap，指定boss和worker EventLoopGroup
    bootstrap.group(bossGroup, workerGroup)
            .channel(NettyEventLoopFactory.serverSocketChannelClass())
            .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
            .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
            .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    // 连接空闲超时时间
                    int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                    // NettyCodecAdapter中会创建Decoder和Encoder
                    NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                    ch.pipeline()
                            // 注册Decoder和Encoder
                            .addLast("decoder", adapter.getDecoder())
                            .addLast("encoder", adapter.getEncoder())
                            // 注册IdleStateHandler
                            .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                            // 注册NettyServerHandler
                            .addLast("handler", nettyServerHandler);
                }
            });
    // 绑定指定的地址和端口
    ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    channelFuture.syncUninterruptibly(); // 等待bind操作完成
    channel = channelFuture.channel();
}

```

NettyServer 实现的 doOpen() 方法的基本流程为: 初始化 ServerBootstrap、创建 Boss EventLoopGroup 和 Worker EventLoopGroup、创建 ChannelInitializer 指定如何初始化 Channel 上的 ChannelHandler 等一系列 Netty 使用的标准化流程.<br>

在 Transporter 这一层看,功能的不同其实就是注册在 Channel 上的 ChannelHandler 不同,通过 doOpen() 方法得到的 Server 端结构如下
[![svyEJP.png](https://s3.ax1x.com/2021/01/27/svyEJP.png)](https://imgchr.com/i/svyEJP)


<br>
**<font size="3">核心 ChannelHandler</font>** <br>

下面来逐个看看上图中这四个 ChannelHandler 的核心功能<br>

首先是decoder和encoder,它们都是NettyCodecAdapter的内部类,如下图所示,分别继承Netty中的ByteToMessageDecoder和MessageToByteEncoder
[![sxkvhF.png](https://s3.ax1x.com/2021/01/27/sxkvhF.png)](https://imgchr.com/i/sxkvhF)

还记得 AbstractEndpoint 抽象类中的 codec 字段(Codec2 类型)吗?InternalDecoder 和 InternalEncoder 会将真正的编解码功能委托给 NettyServer 关联的这个 Codec2 对象去处理,这里以 InternalDecoder 为例进行分析

```
private class InternalDecoder extends ByteToMessageDecoder {
    protected void decode(ChannelHandlerContext ctx, ByteBuf input, List<Object> out) throws Exception {
        // 将ByteBuf封装成统一的ChannelBuffer
        ChannelBuffer message = new NettyBackedChannelBuffer(input);
        // 拿到关联的Channel
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        do {
            // 记录当前readerIndex的位置
            int saveReaderIndex = message.readerIndex();
            // 委托给Codec2进行解码
            Object msg = codec.decode(channel, message);
            // 当前接收到的数据不足一个消息的长度，会返回NEED_MORE_INPUT，
            // 这里会重置readerIndex，继续等待接收更多的数据
            if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                message.readerIndex(saveReaderIndex);
                break;
            } else {
                if (msg != null) { // 将读取到的消息传递给后面的Handler处理
                    out.add(msg);
                }
            }
        } while (message.readable());
    }
}
```
InternalEncoder 的具体实现类似,就不再具体展开了.<br>

接下来是IdleStateHandler,它是 Netty 提供的一个工具型 ChannelHandler,用于定时心跳请求的功能或是自动关闭长时间空闲连接的功能.它的原理是在 IdleStateHandler 中通过 lastReadTime、lastWriteTime 等几个字段,记录了最近一次读/写事件的时间,IdleStateHandler 初始化的时候,会创建一个定时任务,定时检测当前时间与最后一次读/写时间的差值.如果超过我们设置的阈值(也就是上面 NettyServer 中设置的 idleTimeout),就会触发 IdleStateEvent 事件,并传递给后续的 ChannelHandler 进行处理.后续 ChannelHandler 的 userEventTriggered() 方法会根据接收到的 IdleStateEvent 事件,决定是关闭长时间空闲的连接,还是发送心跳探活<br>

最后来看NettyServerHandler,它继承了 ChannelDuplexHandler,这是 Netty 提供的一个同时处理 Inbound 数据和 Outbound 数据的 ChannelHandler,从下面的继承图就能看出来
[![sx3Ydg.png](https://s3.ax1x.com/2021/01/27/sx3Ydg.png)](https://imgchr.com/i/sx3Ydg)

在 NettyServerHandler 中有 channels 和 handler 两个核心字段
* channels(Map<String,Channel>集合):记录了当前 Server 创建的所有 Channel,连接创建(触发 channelActive() 方法)、连接断开(触发 channelInactive()方法)会操作 channels 集合进行相应的增删
* handler(ChannelHandler 类型):NettyServerHandler 内几乎所有方法都会触发该 Dubbo ChannelHandler 对象

在 NettyServer 创建 NettyServerHandler 的时候,可以看到下面的这行代码
```
final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
```
其中第二个参数传入的是 NettyServer 这个对象,你可以追溯一下 NettyServer 的继承结构,会发现它的最顶层父类 AbstractPeer 实现了 ChannelHandler,并且将所有的方法委托给其中封装的 ChannelHandler 对象,也就是说,NettyServerHandler 会将数据委托给这个 ChannelHandler<br>

到此为止,Server 这条继承线就介绍完了.回顾一下,从 AbstractPeer 开始往下,一路继承下来,NettyServer 拥有了 Endpoint、ChannelHandler 以及RemotingServer多个接口的能力,关联了一个 ChannelHandler 对象以及 Codec2 对象,并最终将数据委托给这两个对象进行处理.所以,上层调用方只需要实现 ChannelHandler 和 Codec2 这两个接口就可以了.

[![sx8K7F.png](https://s3.ax1x.com/2021/01/27/sx8K7F.png)](https://imgchr.com/i/sx8K7F)