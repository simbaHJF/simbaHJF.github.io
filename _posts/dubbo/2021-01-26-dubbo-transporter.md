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
<br>
[四. Channel 继承线分析](#jump4)
<br>
[五. ChannelHandler 继承线分析](#jump5)


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
* executorRepository(ExecutorRepository 类型):负责管理线程池
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

<font color="red">NettyServerHandler内部持有了一个ChannelHandler(dubbo语义下定义的ChannelHandler,NettyServer对象实现了该接口), 同时实现了 原生netty语义下定义的ChannelInboundHandler和ChannelOutboundHandler(这两者都集成netty语义下的ChannelHandler接口)接口,这里的实现方式就是通过操作内部持有的dubbo语义下的ChannelHandler来完成.而dubbo语义下的ChannelHandler也定义了一套自己的,不同于netty的事件方法:connected、disconnected、sent、received、caught.因此这里有一个对应方法的匹配,例如实现channelActive方法时采用的是调用内部dubbo语义下ChannelHandler的connected方法,实现channelRead方法时采用的是调用内部dubbo语义下ChannelHandler的received方法等.</font>

到此为止,Server 这条继承线就介绍完了.回顾一下,从 AbstractPeer 开始往下,一路继承下来,NettyServer 拥有了 Endpoint、ChannelHandler 以及RemotingServer多个接口的能力,关联了一个 ChannelHandler 对象以及 Codec2 对象,并最终将数据委托给这两个对象进行处理.所以,上层调用方只需要实现 ChannelHandler 和 Codec2 这两个接口就可以了.

[![sx8K7F.png](https://s3.ax1x.com/2021/01/27/sx8K7F.png)](https://imgchr.com/i/sx8K7F)



<br><br>
## <span id="jump3">三. Client 继承路线分析</span>

接下来,来分析下AbstractClient类,先来开下AbstractClient的继承关系:
[![sz5S9e.png](https://s3.ax1x.com/2021/01/27/sz5S9e.png)](https://imgchr.com/i/sz5S9e)

AbstractClient 中的核心字段有如下几个:
* connectLock(Lock 类型):在 Client 底层进行连接、断开、重连等操作时,需要获取该锁进行同步
* needReconnect(Boolean 类型):在发送数据之前,会检查 Client 底层的连接是否断开,如果断开了,则会根据 needReconnect 字段,决定是否重连
* executor(ExecutorService 类型):当前 Client 关联的线程池

在 AbstractClient 的构造方法中,会解析 URL 初始化 needReconnect 字段和 executor字段,如下示例代码:
```
public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler); // 调用父类的构造方法
    // 解析URL，初始化needReconnect值
    needReconnect = url.getParameter("send.reconnect", false);
    initExecutor(url);     // 解析URL，初始化executor
    doOpen();    // 初始化底层的NIO库的相关组件
    // 创建底层连接
    connect(); // 省略异常处理的逻辑
}

```

与 AbstractServer 类似,AbstractClient 定义了 doOpen()、doClose()、doConnect()和doDisConnect() 四个抽象方法给子类实现.<br>

下面来看基于 Netty 4 实现的 NettyClient,它继承了 AbstractClient 抽象类,实现了上述四个 do\*() 抽象方法,我们这里重点关注 doOpen() 方法和 doConnect() 方法.在 NettyClient 的 doOpen() 方法中会通过 Bootstrap 构建客户端,其中会完成连接超时时间、keepalive 等参数的设置,以及 ChannelHandler 的创建和注册,具体实现如下所示:
```
protected void doOpen() throws Throwable {
    // 创建NettyClientHandler
    final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
    bootstrap = new Bootstrap(); // 创建Bootstrap
    bootstrap.group(NIO_EVENT_LOOP_GROUP)
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .channel(socketChannelClass());
    // 设置连接超时时间，这里使用到AbstractEndpoint中的connectTimeout字段
    bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, Math.max(3000, getConnectTimeout()));
    bootstrap.handler(new ChannelInitializer<SocketChannel>() {
        protected void initChannel(SocketChannel ch) throws Exception {
            // 心跳请求的时间间隔
            int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
            // 通过NettyCodecAdapter创建Netty中的编解码器
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
            // 注册ChannelHandler
            ch.pipeline().addLast("decoder", adapter.getDecoder())
                    .addLast("encoder", adapter.getEncoder())
                    .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                    .addLast("handler", nettyClientHandler);
            // 如果需要Socks5Proxy，需要添加Socks5ProxyHandler(略)
        }
    });
}

```
得到的 NettyClient 结构如下图所示:
[![szIma6.png](https://s3.ax1x.com/2021/01/27/szIma6.png)](https://imgchr.com/i/szIma6)

NettyClientHandler 的实现方法与 NettyServerHandler 类似,同样是实现了 Netty 中的 ChannelDuplexHandler,其中会将所有方法委托给 NettyClient 关联的 ChannelHandler 对象进行处理.两者在 userEventTriggered() 方法的实现上有所不同,NettyServerHandler 在收到 IdleStateEvent 事件时会断开连接,而 NettyClientHandler 则会发送心跳消息,具体实现如下:
```
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    if (evt instanceof IdleStateEvent) {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        Request req = new Request(); 
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setEvent(HEARTBEAT_EVENT); // 发送心跳请求
        channel.send(req);
    } else {
        super.userEventTriggered(ctx, evt);
    }
}
```


<br><br>
## <span id="jump4">四. Channel 继承线分析</span>

除了前面介绍的 AbstractEndpoint 之外,AbstractChannel 也继承了 AbstractPeer 这个抽象类,同时还继承了 Channel 接口.AbstractChannel 实现非常简单,只是在 send() 方法中检测了底层连接的状态,没有实现具体的发送消息的逻辑.<br>

这里我们依然以基于 Netty 4 的实现—— NettyChannel 为例,分析它对 AbstractChannel 的实现.NettyChannel 中的核心字段有如下几个
* channel(Channel类型):Netty 框架中的 Channel,与当前的 Dubbo Channel 对象一一对应.
* attributes(Map<String, Object>类型):当前 Channel 中附加属性,都会记录到该 Map 中.NettyChannel 中提供的 getAttribute()、hasAttribute()、setAttribute() 等方法,都是操作该集合
* active(AtomicBoolean):用于标识当前 Channel 是否可用

另外,在 NettyChannel 中还有一个静态的 Map 集合(CHANNEL_MAP 字段),用来缓存当前 JVM 中 Netty 框架 Channel 与 Dubbo Channel 之间的映射关系.<br>

NettyChannel 中还有一个要介绍的是 send() 方法,它会通过底层关联的 Netty 框架 Channel,将数据发送到对端.其中,可以通过第二个参数指定是否等待发送操作结束,具体实现如下:
```
public void send(Object message, boolean sent) throws RemotingException {
	// whether the channel is closed
	// 调用AbstractChannel的send()方法检测连接是否可用
	super.send(message, sent);
	boolean success = true;
	int timeout = 0;
	try {
		// 依赖Netty框架的Channel发送数据
	    ChannelFuture future = channel.writeAndFlush(message);
	    if (sent) { // 等待发送结束，有超时时间
	        // wait timeout ms
	        timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
	        success = future.await(timeout);
	    }
	    Throwable cause = future.cause();
	    if (cause != null) {
	        throw cause;
	    }
	} catch (Throwable e) {
		// 出现异常会调用removeChannelIfDisconnected()方法,在底层连接断开时,会清理CHANNEL_MAP缓存
	    removeChannelIfDisconnected(channel);
	    throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + 	" to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
	}
	if (!success) {
	    throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + 	" to " + getRemoteAddress()
	            + "in timeout(" + timeout + "ms) limit");
	}
}
```



<br><br>
## <span id="jump5">五. ChannelHandler 继承线分析</span>

前文介绍的 AbstractServer、AbstractClient 以及 Channel 实现,都是通过 AbstractPeer 实现了 ChannelHandler 接口,但只是做了一层简单的委托(也可以说成是装饰器),将全部方法委托给了其底层关联的 ChannelHandler 对象.<br>

这里我们就深入分析 ChannelHandler 的其他实现类,涉及的实现类如下所示:
[![y9vt1A.png](https://s3.ax1x.com/2021/01/28/y9vt1A.png)](https://imgchr.com/i/y9vt1A)

<font color="red">ChannelHandlerDispatcher是负责将多个ChannelHandler对象聚合成一个ChannelHandler对象.</font>

ChannelHandlerAdapter是 ChannelHandler 的一个空实现,TelnetHandlerAdapter 继承了它并实现了 TelnetHandler 接口.<br>

ChannelHandlerDelegate接口是对另一个 ChannelHandler 对象的封装,它的两个实现类 AbstractChannelHandlerDelegate 和 WrappedChannelHandler 中也仅仅是封装了另一个 ChannelHandler 对象.<br>


<br>
**<font size="3">ChannelHandlerDelegate ----> AbstractChannelHandlerDelegate 继承线</font>** <br>

首先来看下AbstractChannelHandlerDelegate,它有三个实现类:
* MultiMessageHandler: 专门处理 MultiMessage 的 ChannelHandler 实现.MultiMessage 是 Exchange 层的一种消息类型,它其中封装了多个消息.在 MultiMessageHandler 收到 MultiMessage 消息的时候,received() 方法会遍历其中的所有消息,并交给底层的 ChannelHandler 对象进行处理
* DecodeHandler: 专门处理 Decodeable 的 ChannelHandler 实现.实现了 Decodeable 接口的类都会提供了一个 decode() 方法实现对自身的解码,DecodeHandler.received() 方法就是通过该方法得到解码后的消息,然后传递给底层的 ChannelHandler 对象继续处理
* HeartbeatHandler: 专门处理心跳消息的 ChannelHandler 实现.在 HeartbeatHandler.received() 方法接收心跳请求的时候,会生成相应的心跳响应并返回;在收到心跳响应的时候,会打印相应的日志;在收到其他类型的消息时,会传递给底层的 ChannelHandler 对象进行处理.下面是其核心实现:
```
public void received(Channel channel, Object message) throws RemotingException {
    setReadTimestamp(channel); // 记录最近的读写事件时间戳
    if (isHeartbeatRequest(message)) { // 收到心跳请求
        Request req = (Request) message;
        if (req.isTwoWay()) { // 返回心跳响应，注意，携带请求的ID
            Response res = new Response(req.getId(), req.getVersion());
            res.setEvent(HEARTBEAT_EVENT);
            channel.send(res);
        return;
    }
    if (isHeartbeatResponse(message)) { // 收到心跳响应
        // 打印日志(略)
        return;
    }
    handler.received(channel, message);
}
```
另外我们可以看到,在 received() 和 send() 方法中,HeartbeatHandler 会将最近一次的读写时间作为附加属性记录到 Channel 中.<br>

AbstractChannelHandlerDelegate 下的三个实现,其实都是在原有 ChannelHandler 的基础上添加了一些增强功能,这是典型的装饰器模式的应用


<br>
**<font size="3">ChannelHandlerDelegate ----> WrappedChannelHandler 继承线</font>** <br>

<font color="red">WrappedChannelHandler子类主要是决定了 Dubbo 以何种线程模型处理收到的事件和消息,就是所谓的“消息派发机制”,与前面介绍的 ThreadPool 有紧密的联系</font>
[![yCCshT.png](https://s3.ax1x.com/2021/01/28/yCCshT.png)](https://imgchr.com/i/yCCshT)

从上图中我们可以看到,每个 WrappedChannelHandler 实现类的对象都由一个相应的 Dispatcher 实现类创建,下面是 Dispatcher 接口的定义
```
@SPI(AllDispatcher.NAME) // 默认扩展名是all
public interface Dispatcher {
    // 通过URL中的参数可以指定扩展名，覆盖默认扩展名
    @Adaptive({"dispatcher", "dispather", "channel.handler"})
    ChannelHandler dispatch(ChannelHandler handler, URL url);
}
```

AllDispatcher 创建的是 AllChannelHandler 对象,它会将所有网络事件以及消息交给关联的线程池进行处理.

<font color="red">官方文档说:"all 模式是所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等".这里我认为写的是有误的,心跳事件不会被派发到线程池而是在IO线程直接就搞定返回了.分析源码查看各个ChannelHandler的包装顺序也可以发现,处理心跳的ChannelHandler是在线程模型ChannelHandler之前就执行完了的.</font>
**<font color="red">换句话总结说,dubbo中设置的任何一种线程模型,从源码分析角度,都不可能把心跳消息派发到业务线程池中.</font>**

AllChannelHandler覆盖了 WrappedChannelHandler 中除了 sent() 方法之外的其他网络事件处理方法,将调用其底层的 ChannelHandler 的逻辑放到关联的线程池中执行.<br>

我们先来看 connect() 方法,其中会将CONNECTED 事件的处理封装成ChannelEventRunnable提交到线程池中执行,具体实现如下:
```
public void connected(Channel channel) throws RemotingException {
    ExecutorService executor = getExecutorService(); // 获取公共线程池
    // 将CONNECTED事件的处理封装成ChannelEventRunnable提交到线程池中执行
    executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
    // 省略异常处理的逻辑
}
```

这里的 getExecutorService() 方法会按照当前端点(Server/Client)的 URL 从 ExecutorRepository 中获取相应的公共线程池.<br>

disconnected()方法处理连接断开事件,caught() 方法处理异常事件,它们也是按照上述方式实现的,这里不再展开赘述.<br>

received() 方法会在当前端点收到数据的时候被调用,用 AllChannelHandler 的 received() 方法,其中会将请求提交给线程池执行,执行完后调用 send()方法,向对端写回响应结果.received()方法的具体实现如下:
```
public void received(Channel channel, Object message) throws RemotingException {
	// 获取线程池
    ExecutorService executor = getPreferredExecutorService(message);
    try {
    	// 将消息封装成ChannelEventRunnable任务，提交到线程池中执行
        executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
    } catch (Throwable t) {
    	// 如果线程池满了，请求会被拒绝，这里会根据请求配置决定是否返回一个说明性的响应
    	if(message instanceof Request && t instanceof RejectedExecutionException){
            sendFeedback(channel, (Request) message, t);
            return;
    	}
        throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
    }
}
```


ExecutionChannelHandler(由 ExecutionDispatcher 创建)只会将请求消息派发到线程池进行处理,也就是只重写了 received() 方法.对于响应消息以及其他网络事件(例如,连接建立事件、连接断开事件、心跳消息等),ExecutionChannelHandler 会直接在 IO 线程中进行处理.<br>

DirectChannelHandler 实现(由 DirectDispatcher 创建)会在 IO 线程中处理所有的消息和网络事件.<br>

MessageOnlyChannelHandler 实现(由 MessageOnlyDispatcher 创建)会将所有收到的消息提交到线程池处理,其他网络事件则是由 IO 线程直接处理.<br>

ConnectionOrderedChannelHandler 实现(由 ConnectionOrderedDispatcher 创建)会将收到的消息交给线程池进行处理,对于连接建立以及断开事件,会提交到一个独立的线程池并排队进行处理.<br>



到此为止,Transporter 层对 ChannelHandler 的实现就介绍完了,其中涉及了多个 ChannelHandler 的装饰器,为了帮助你更好地理解,这里我们回到 NettyServer 中,看看它是如何对上层 ChannelHandler 进行封装的.<br>

在 NettyServer 的构造方法中会调用 ChannelHandlers.wrap() 方法对传入的 ChannelHandler 对象进行修饰:
```
protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
    return new MultiMessageHandler(new HeartbeatHandler(ExtensionLoader.getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
}
```


结合前面的分析，我们可以得到下面这张图:
[![yCKaGj.png](https://s3.ax1x.com/2021/01/29/yCKaGj.png)](https://imgchr.com/i/yCKaGj)

我们可以在创建 NettyServerHandler 的地方添加断点 Debug 得到下图,也印证了上图的内容:
[![yCK2i4.png](https://s3.ax1x.com/2021/01/29/yCK2i4.png)](https://imgchr.com/i/yCK2i4)