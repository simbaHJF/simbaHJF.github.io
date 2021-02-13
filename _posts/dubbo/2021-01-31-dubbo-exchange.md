---
layout:     post
title:      "Dubbo Remoting Exchange层"
date:       2021-01-31 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---


## 导航
[一. 前言](#jump1)
<br>
[二. Request 和 Response](#jump2)
<br>
[三. ExchangeChannel 和 DefaultFuture](#jump3)
<br>
[四. HeaderExchangeHandler](#jump4)
<br>
[五. HeaderExchangeClient](#jump5)
<br>
[六. HeaderExchangeServer](#jump6)
<br>
[七. HeaderExchanger](#jump7)
<br>
[八. 再谈 Codec2](#jump8)
<br>
[九. 一点总结](#jump9)



<br><br>
## <span id="jump1">一. 前言</span>
Dubbo 将信息交换行为抽象成 Exchange 层,官方文档对这一层的说明是:封装了请求-响应的语义,即关注一问一答的交互模式,实现了同步转异步.在 Exchange 这一层,以 Request 和 Response 为中心,针对 Channel、ChannelHandler、Client、RemotingServer 等接口进行实现.<br>
<font color="red">这些接口的定义都是在remoting层,而在exchange做了一些实现,体现了下层进行抽象定义,上层调用方根据使用需要做具体细节实现的这样一种思想</font>


<br><br>
## <span id="jump2">二. Request 和 Response</span>

Exchange 层的 Request 和 Response 这两个类是 Exchange 层的核心对象,是对请求和响应的抽象.
<font color="red">Request 和 Response并不是被底层传输的对象,而只是Exchange层抽象出来的在端点内进行一些操作用到的数据结构,不参与网络传输.网络传输的数据是根据Request 和 Response及其他一些信息构建出来的消息头、消息体的字节数组序列（这个过程涉及正反序列化）,而不是直接对Request 和 Response进行正反序列化</font>

我们先来看Request 类的核心字段:
```
public class Request {
    // 用于生成请求的自增ID，当递增到Long.MAX_VALUE之后，会溢出到Long.MIN_VALUE，我们可以继续使用该负数作为消息ID
    private static final AtomicLong INVOKE_ID = new AtomicLong(0);

	// 请求的ID
    private final long mId;

	// 请求版本号
    private String mVersion;

    // 请求的双向标识，如果该字段设置为true，则Server端在收到请求后，
    // 需要给Client返回一个响应
    private boolean mTwoWay = true;

    // 事件标识，例如心跳请求、只读请求等，都会带有这个标识
    private boolean mEvent = false;

    // 请求发送到Server之后，由Decoder将二进制数据解码成Request对象，
    // 如果解码环节遇到异常，则会设置该标识，然后交由其他ChannelHandler根据
    // 该标识做进一步处理
    private boolean mBroken = false;

    // 请求体，可以是任何Java类型的对象,也可以是null
    private Object mData;
}
```

接下来是 Response 的核心字段:
```
public class Response {

    // 响应ID，与相应请求的ID一致
    private long mId = 0;

    // 当前协议的版本号，与请求消息的版本号一致
    private String mVersion;

    // 响应状态码，有OK、CLIENT_TIMEOUT、SERVER_TIMEOUT等10多个可选值
    private byte mStatus = OK; 

    private boolean mEvent = false;

	// 可读的错误响应消息
    private String mErrorMsg;

	// 响应体
    private Object mResult;
}
```



<br><br>
## <span id="jump3">三. ExchangeChannel 和 DefaultFuture</span>

Exchange 层中定义了 ExchangeChannel 接口,它在 Channel 接口之上抽象了 Exchange 层的网络连接.ExchangeChannel 接口的定义如下:
[![yEeaUf.png](https://s3.ax1x.com/2021/01/31/yEeaUf.png)](https://imgchr.com/i/yEeaUf)

其中,request() 方法负责发送请求,从图中可以看到这里有两个重载,其中一个重载可以指定请求的超时时间,返回值都是 Future 对象.

[![yEmVJS.png](https://s3.ax1x.com/2021/01/31/yEmVJS.png)](https://imgchr.com/i/yEmVJS)
HeaderExchangeChannel 是 ExchangeChannel 的实现,它本身是 Channel 的装饰器,封装了一个 Channel 对象,其 send() 方法和 request() 方法的实现都是依赖底层修饰的这个 Channel 对象实现的

```
public void send(Object message, boolean sent) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send message " + message + ", cause: The channel " + this + " is closed!");
    }
    if (message instanceof Request
            || message instanceof Response
            || message instanceof String) {
        channel.send(message, sent);
    } else {
        Request request = new Request();
        request.setVersion(Version.getProtocolVersion());
        request.setTwoWay(false);
        request.setData(message);
        channel.send(request, sent);
    }
}



public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
    try {
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

这里的 request() 方法,它返回的是一个 DefaultFuture 对象,DefaultFuture 就表示此次请求-响应是否完成,也就是说,要收到响应为 Future 才算完成.<br>

下面我们就来深入介绍一下请求发送过程中涉及的 DefaultFuture 以及HeaderExchangeChannel的内容<br>

首先来了解一下 DefaultFuture 的具体实现,它继承了 JDK 中的 CompletableFuture,其中维护了两个 static 集合
* CHANNELS(Map<Long, Channel>集合): 管理请求与 Channel 之间的关联关系,其中 Key 为请求 ID,Value 为发送请求的 Channel
* FUTURES(Map<Long, Channel>集合): 管理请求与 DefaultFuture 之间的关联关系,其中 Key 为请求 ID,Value 为请求对应的 Future

DefaultFuture 中核心的实例字段包括如下几个:
* request(Request 类型)和 id(Long 类型): 对应请求以及请求的 ID
* channel(Channel 类型): 发送请求的 Channel
* timeout(int 类型): 整个请求-响应交互完成的超时时间
* start(long 类型): 该 DefaultFuture 的创建时间
* sent(volatile long 类型): 请求发送的时间
* timeoutCheckTask(Timeout 类型): 该定时任务到期时，表示对端响应超时
* executor(ExecutorService 类型): 请求关联的线程池

DefaultFuture.newFuture() 方法创建 DefaultFuture 对象时,需要先初始化上述字段,并创建请求相应的超时定时任务
```
public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {

    // 创建DefaultFuture对象，并初始化其中的核心字段
    final DefaultFuture future = new DefaultFuture(channel, request, timeout);
    future.setExecutor(executor); 
    // 对于ThreadlessExecutor的特殊处理，ThreadlessExecutor可以关联一个waitingFuture，就是这里创建DefaultFuture对象
    if (executor instanceof ThreadlessExecutor) {
        ((ThreadlessExecutor) executor).setWaitingFuture(future);
    }
    // 创建一个定时任务,用于处理响应超时的情况
    timeoutCheck(future);
    return future;
}
```

在 HeaderExchangeChannel.request() 方法中完成 DefaultFuture 对象的创建之后,会将请求通过底层的 Dubbo Channel 发送出去,发送过程中会触发沿途 ChannelHandler 的 sent() 方法,其中的 HeaderExchangeHandler 会调用 DefaultFuture.sent() 方法更新 sent 字段,记录请求发送的时间戳.后续如果响应超时,则会将该发送时间戳添加到提示信息中.<br>

过一段时间之后,Consumer 会收到对端返回的响应,在读取到完整响应之后,会触发 Dubbo Channel 中各个 ChannelHandler 的 received() 方法,其中就包括之前介绍的 WrappedChannelHandler.例如,AllChannelHandler 子类会将后续 ChannelHandler.received() 方法的调用封装成任务提交到线程池中,响应会提交到 DefaultFuture 关联的线程池中,然后由业务线程继续后续的 ChannelHandler 调用.<br>

当响应传递到 HeaderExchangeHandler 的时候,会通过调用 handleResponse() 方法进行处理,其中调用了 DefaultFuture.received() 方法,该方法会找到响应关联的 DefaultFuture 对象(根据请求 ID 从 FUTURES 集合查找)并调用 doReceived() 方法,将 DefaultFuture 设置为完成状态.
```
public static void received(Channel channel, Response response, boolean timeout) { // 省略try/finally代码块

    // 清理FUTURES中记录的请求ID与DefaultFuture之间的映射关系
    DefaultFuture future = FUTURES.remove(response.getId()); 
    if (future != null) {
        Timeout t = future.timeoutCheckTask;
        if (!timeout) { 
			// 未超时，取消定时任务
            t.cancel();
        }
        future.doReceived(response); // 调用doReceived()方法
    }else{ 
		// 查找不到关联的DefaultFuture会打印日志(略)
	}
    // 清理CHANNELS中记录的请求ID与Channel之间的映射关系
    CHANNELS.remove(response.getId()); 
}


// DefaultFuture.doReceived()方法的代码片段
private void doReceived(Response res) {
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) { 
		// 正常响应
        this.complete(res.getResult());
    } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) { 
			// 超时
            this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
    } else { 
			// 其他异常
            this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
    }
    // 下面是针对ThreadlessExecutor的兜底处理，主要是防止业务线程一直阻塞在ThreadlessExecutor上
    if (executor != null && executor instanceof ThreadlessExecutor) {
        ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
        if (threadlessExecutor.isWaiting()) {
            // notifyReturn()方法会向ThreadlessExecutor提交一个任务，这样业务线程就不会阻塞了，提交的任务会尝试将DefaultFuture设置为异常结束
            threadlessExecutor.notifyReturn(new IllegalStateException("The result has returned..."));
        }
    }
}
```

下面我们再来看看响应超时的场景.在创建 DefaultFuture 时调用的 timeoutCheck() 方法中,会创建 TimeoutCheckTask 定时任务,并添加到时间轮中,具体实现如下
```
private static void timeoutCheck(DefaultFuture future) {	
    TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
    future.timeoutCheckTask = TIME_OUT_TIMER.newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
}
```

TIME_OUT_TIMER 是一个 HashedWheelTimer 对象,即 Dubbo 中对时间轮的实现,这是一个 static 字段,所有 DefaultFuture 对象共用一个.<br>

TimeoutCheckTask 是 DefaultFuture 中的内部类,实现了 TimerTask 接口,可以提交到时间轮中等待执行.当响应超时的时候,TimeoutCheckTask 会创建一个 Response,并调用前面介绍的 DefaultFuture.received() 方法.示例代码如下
```
public void run(Timeout timeout) {
    // 检查该任务关联的DefaultFuture对象是否已经完成
    if (future.getExecutor() != null) { // 提交到线程池执行，注意ThreadlessExecutor的情况
        future.getExecutor().execute(() -> notifyTimeout(future));
    } else {
        notifyTimeout(future);
    }
}

private void notifyTimeout(DefaultFuture future) {
    // 没有收到对端的响应，这里会创建一个Response，表示超时的响应
    Response timeoutResponse = new Response(future.getId());
    timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
    timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
    // 将关联的DefaultFuture标记为超时异常完成
    DefaultFuture.received(future.getChannel(), timeoutResponse, true);
}
```

<font color="red">这里总结一下,正常的response返回以及提交到时间轮中的超时检查任务,最终都会调用到DefaultFuture.received()方法,在该方法中首先会从FUTURES中remove请求id对应的DefaultFuture,判断remove结果不为空才会继续执行,以此保证并发时的安全性</font>


<br><br>
## <span id="jump4">四. HeaderExchangeHandler</span>

在前面介绍 DefaultFuture 时,我们简单说明了请求-响应的流程,其实无论是发送请求还是处理响应,都会涉及 HeaderExchangeHandler,所以这里我们就来介绍一下 HeaderExchangeHandler 的内容.<br>

HeaderExchangeHandler 是 ExchangeHandler 的装饰器,其中维护了一个 ExchangeHandler 对象,ExchangeHandler 接口是 Exchange 层与上层交互的接口之一,上层调用方可以实现该接口完成自身的功能;然后再由 HeaderExchangeHandler 修饰,具备 Exchange 层处理 Request-Response 的能力;最后再由 Transport ChannelHandler 修饰,具备 Transport 层的能力.<br>

HeaderExchangeHandler 作为一个装饰器,其 connected()、disconnected()、sent()、received()、caught() 方法最终都会转发给上层提供的 ExchangeHandler 进行处理.这里我们需要聚焦的是 HeaderExchangeHandler 本身对 Request 和 Response 的处理逻辑<br>
[![yVizZT.png](https://s3.ax1x.com/2021/01/31/yVizZT.png)](https://imgchr.com/i/yVizZT)

结合上图,我们可以看到在received() 方法中,对收到的消息进行了分类处理
* 只读请求会由handlerEvent() 方法进行处理,它会在 Channel 上设置 channel.readonly 标志,后续介绍的上层调用中会读取该值
```
void handlerEvent(Channel channel, Request req) throws RemotingException {
    if (req.getData() != null && req.getData().equals(READONLY_EVENT)) {
        channel.setAttribute(Constants.CHANNEL_ATTRIBUTE_READONLY_KEY, Boolean.TRUE);
    }
}
```
* 双向请求由handleRequest() 方法进行处理,会先对解码失败的请求进行处理,返回异常响应;然后将正常解码的请求交给上层实现的 ExchangeHandler 进行处理.并添加回调.上层 ExchangeHandler 处理完请求后,会触发回调,根据处理结果填充响应结果和响应码,并向对端发送
```
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    if (req.isBroken()) { // 请求解码失败
        Object data = req.getData();
        // 设置异常信息和响应码
        res.setErrorMessage("Fail to decode request due to: " + msg);
        res.setStatus(Response.BAD_REQUEST); 
        channel.send(res); // 将异常响应返回给对端
        return;
    }
    Object msg = req.getData();
    // 交给上层实现的ExchangeHandler进行处理
    CompletionStage<Object> future = handler.reply(channel, msg);
    future.whenComplete((appResult, t) -> { // 处理结束后的回调
        if (t == null) { // 返回正常响应
            res.setStatus(Response.OK);
            res.setResult(appResult);
        } else { // 处理过程发生异常，设置异常信息和错误码
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(t));
        }
        channel.send(res); // 发送响应
    });
}
```
* 单向请求直接委托给上层 ExchangeHandler 实现的 received() 方法进行处理,由于不需要响应,HeaderExchangeHandler 不会关注处理结果
* 对于 Response 的处理,前文已提到了,HeaderExchangeHandler 会通过handleResponse() 方法将关联的 DefaultFuture 设置为完成状态(或是异常完成状态),具体内容这里不再展开讲述
* 对于 String 类型的消息,HeaderExchangeHandler 会根据当前服务的角色进行分类,具体与 Dubbo 对 telnet 的支持相关,后面会详细介绍,这里就不展开分析了

接下来我们再来看sent() 方法,该方法会通知上层 ExchangeHandler 实现的 sent() 方法,同时还会针对 Request 请求调用 DefaultFuture.sent() 方法记录请求的具体发送时间<br>

在connected() 方法中,会为 Dubbo Channel 创建相应的 HeaderExchangeChannel,并将两者绑定,然后通知上层 ExchangeHandler 处理 connect 事件<br>

在disconnected() 方法中,首先通知上层 ExchangeHandler 进行处理,之后在 DefaultFuture.closeChannel() 通知 DefaultFuture 连接断开(其实就是创建并传递一个 Response,该 Response 的状态码为 CHANNEL_INACTIVE),这样就不会继续阻塞业务线程了,最后再将 HeaderExchangeChannel 与底层的 Dubbo Channel 解绑.<br>



<br><br>
## <span id="jump5">五. HeaderExchangeClient</span>

HeaderExchangeClient 是 Client 装饰器,主要为其装饰的 Client 添加两个功能:
* 维持与 Server 的长连状态,这是通过定时发送心跳消息实现的
* 在因故障掉线之后,进行重连,这是通过定时检查连接状态实现的

因此,HeaderExchangeClient 侧重定时轮资源的分配、定时任务的创建和取消<br>

HeaderExchangeClient 实现的是 ExchangeClient 接口,如下图所示,间接实现了 ExchangeChannel 和 Client 接口,ExchangeClient 接口是个空接口,没有定义任何方法
[![yuI1XQ.png](https://s3.ax1x.com/2021/02/02/yuI1XQ.png)](https://imgchr.com/i/yuI1XQ)

HeaderExchangeClient 中有以下两个核心字段:
* client(Client 类型):被修饰的 Client 对象.HeaderExchangeClient 中对 Client 接口的实现,都会委托给该对象进行处理
* channel(ExchangeChannel 类型):Client 与服务端建立的连接,HeaderExchangeClient 中对 ExchangeChannel 接口的实现,都会委托给该对象进行处理.<br>

HeaderExchangeClient 构造方法的第一个参数封装 Transport 层的 Client 对象,第二个参数 startTimer参与控制是否开启心跳定时任务和重连定时任务,如果为 true,才会进一步根据其他条件,最终决定是否启动定时任务.这里我们以心跳定时任务为例:
```
private void startHeartBeatTask(URL url) {

    if (!client.canHandleIdle()) { // Client的具体实现决定是否启动该心跳任务

        AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);

        // 计算心跳间隔，最小间隔不能低于1s
        int heartbeat = getHeartbeat(url); 

        long heartbeatTick = calculateLeastDuration(heartbeat);

        // 创建心跳任务
        this.heartBeatTimerTask = new HeartbeatTimerTask(cp, heartbeatTick, heartbeat);

        // 提交到IDLE_CHECK_TIMER这个时间轮中等待执行
        IDLE_CHECK_TIMER.newTimeout(heartBeatTimerTask, heartbeatTick, TimeUnit.MILLISECONDS);
    }
}
```

重连定时任务是在 startReconnectTask() 方法中启动的,其中会根据 URL 中的参数决定是否启动任务.重连定时任务最终也是提交到 IDLE_CHECK_TIMER 这个时间轮中,时间轮定义如下
```
private static final HashedWheelTimer IDLE_CHECK_TIMER = new HashedWheelTimer(

            new NamedThreadFactory("dubbo-client-idleCheck", true), 1, TimeUnit.SECONDS, TICKS_PER_WHEEL);

```
startReconnectTask() 方法的具体实现与前面展示的 startHeartBeatTask() 方法类似,这里就不再赘述.<br>

下面继续回到心跳定时任务进行分析,在NettyClient 实现中,其 canHandleIdle() 方法返回 true,表示该实现可以自己发送心跳请求,无须 HeaderExchangeClient 再启动一个定时任务.NettyClient 主要依靠 IdleStateHandler 中的定时任务来触发心跳事件,依靠 NettyClientHandler 来发送心跳请求.<br>

对于无法自己发送心跳请求的 Client 实现,HeaderExchangeClient 会为其启动 HeartbeatTimerTask 心跳定时任务,其继承关系如下图所示
[![yuxnNF.png](https://s3.ax1x.com/2021/02/03/yuxnNF.png)](https://imgchr.com/i/yuxnNF)

AbstractTimerTask 这个抽象类,它有三个字段:
* channelProvider(ChannelProvider类型): ChannelProvider 是 AbstractTimerTask 抽象类中定义的内部接口,定时任务会从该对象中获取 Channel
* tick(Long类型): 任务的过期时间
* cancel(boolean类型): 任务是否已取消

AbstractTimerTask 抽象类实现了 TimerTask 接口的 run() 方法,首先会从 ChannelProvider 中获取此次任务相关的 Channel 集合(在 Client 端只有一个 Channel,在 Server 端有多个 Channel),然后检查 Channel 的状态,针对未关闭的 Channel 执行 doTask() 方法处理,最后通过 reput() 方法将当前任务重新加入时间轮中,等待再次到期执行,其run方法的具体实现如下:
```
public void run(Timeout timeout) throws Exception {
    // 从ChannelProvider中获取任务要操作的Channel集合
    Collection<Channel> c = channelProvider.getChannels();
    for (Channel channel : c) {
        if (channel.isClosed()) { // 检测Channel状态
            continue;
        }
        doTask(channel); // 执行任务
    }
    reput(timeout, tick); // 将当前任务重新加入时间轮中，等待执行
}
```

doTask() 是一个 AbstractTimerTask 留给子类实现的抽象方法,不同的定时任务执行不同的操作.例如,HeartbeatTimerTask.doTask() 方法中会读取最后一次读写时间,然后计算距离当前的时间,如果大于心跳间隔,就会发送一个心跳请求,核心实现如下:
```
protected void doTask(Channel channel) {
    // 获取最后一次读写时间
    Long lastRead = lastRead(channel);
    Long lastWrite = lastWrite(channel);
    if ((lastRead != null && now() - lastRead > heartbeat)
            || (lastWrite != null && now() - lastWrite > heartbeat)) {
        // 最后一次读写时间超过心跳时间，就会发送心跳请求
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setEvent(HEARTBEAT_EVENT);
        channel.send(req);
    }
}
```

这里 lastRead 和 lastWrite 时间戳,都是从要待处理 Channel 的附加属性中获取的,对应的 Key 分别是:KEY_READ_TIMESTAMP、KEY_WRITE_TIMESTAMP.前面介绍过的HeartbeatHandler,它属于 Transport 层,是一个 ChannelHandler 的装饰器,在其 connected() 、sent() 方法中会记录最后一次写操作时间,在其 connected()、received() 方法中会记录最后一次读操作时间,在其 disconnected() 方法中会清理这两个时间戳.<br>

在 ReconnectTimerTask 中会检测待处理 Channel 的连接状态,以及读操作的空闲时间,对于断开或是空闲时间较长的 Channel 进行重连,具体逻辑这里就不再展开了.<br>

HeaderExchangeClient 最后要关注的是它的关闭流程,具体实现在 close() 方法中,如下所示:
```
public void close(int timeout) {
    startClose(); // 将closing字段设置为true
    doClose(); // 关闭心跳定时任务和重连定时任务
    channel.close(timeout); // 关闭HeaderExchangeChannel
}
```

在 HeaderExchangeChannel.close(timeout) 方法中首先会将自身的 closed 字段设置为 true,这样就不会继续发送请求.如果当前 Channel 上还有请求未收到响应,会循环等待至收到响应,如果超时未收到响应,会自己创建一个状态码将连接关闭的 Response 交给 DefaultFuture 处理,与收到 disconnected 事件相同.然后会关闭 Transport 层的 Channel,以 NettyChannel 为例,NettyChannel.close() 方法会先将自身的 closed 字段设置为 true,清理 CHANNEL_MAP 缓存中的记录,以及 Channel 的附加属性,最后才是关闭 io.netty.channel.Channel.



<br><br>
## <span id="jump6">六. HeaderExchangeServer</span>

下面再来看 HeaderExchangeServer,其继承关系如下图所示
[![y3hEhn.png](https://s3.ax1x.com/2021/02/04/y3hEhn.png)](https://imgchr.com/i/y3hEhn)

与前面介绍的 HeaderExchangeClient 一样,HeaderExchangeServer 是 RemotingServer 的装饰器,实现自 RemotingServer 接口的大部分方法都委托给了所修饰的 RemotingServer 对象<br>

在 HeaderExchangeServer 的构造方法中,会启动一个 CloseTimerTask 定时任务,定期关闭长时间空闲的连接,具体的实现方式与 HeaderExchangeClient 中的两个定时任务类似,这里不再展开分析.<br>

需要注意的是, NettyServer 并没有启动该定时任务,而是靠 NettyServerHandler 和 IdleStateHandler 实现的,原理与 NettyClient 类似,这里不再展开,可以翻看 CloseTimerTask 源码进行了解.


<br><br>
## <span id="jump7">七. HeaderExchanger</span>

对于上层来说,Exchange 层的入口是 Exchangers 这个门面类,其中提供了多个 bind() 以及 connect() 方法的重载,这些重载方法最终会通过 SPI 机制,获取 Exchanger 接口的扩展实现,这个流程 Transport 层的入口---- Transporters 门面类相同.<br>

Exchanger 接口的定义与 Transporter 接口非常类似,同样是被 @SPI 接口修饰(默认扩展名为"header",对应的是 HeaderExchanger 这个实现),bind() 方法和 connect() 方法也同样是被 @Adaptive 注解修饰,可以通过 URL 参数中的 exchanger 参数值指定扩展名称来覆盖默认值
```
@SPI(HeaderExchanger.NAME)

public interface Exchanger {

    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;

    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;
}
```

Dubbo 只为 Exchanger 接口提供了 HeaderExchanger 这一个实现,其中 connect() 方法创建的是 HeaderExchangeClient 对象,bind() 方法创建的是 HeaderExchangeServer 对象,如下图所示:
[![y34Ozt.png](https://s3.ax1x.com/2021/02/04/y34Ozt.png)](https://imgchr.com/i/y34Ozt)

从 HeaderExchanger 的实现可以看到,它会在 Transport 层的 Client 和 Server 实现基础之上,添加前文介绍的 HeaderExchangeClient 和 HeaderExchangeServer 装饰器.同时，为上层实现的 ExchangeHandler 实例添加了 HeaderExchangeHandler 以及 DecodeHandler 两个修饰器:
```
public class HeaderExchanger implements Exchanger {
    public static final String NAME = "header";
    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
}
```

<font color="red">这里 HeaderExchangeClient 和 HeaderExchangeServer 内部会关联 ExchangeChannel , ExchangeChannel 实现了 request 和 response 语义.</font>



<br><br>
## <span id="jump8">八. 再谈 Codec2</span>

Codec2 接口提供了 encode() 和 decode() 两个方法来实现消息与字节流之间的相互转换.需要注意与 DecodeHandler 区分开来,DecodeHandler 是对请求体和响应结果的解码,Codec2 是对整个请求和响应的编解码.<br>

具体点来说,以decode为例,Codec2按照协议规则(默认dubbo)解码消息头(这里包括对消息TCP半包和粘包的处理),将消息体数据流(ChannelBufferInputStream) 封装成 DecodeableRpcResult , 然后根据Channel持有的url中的"decode.in.io"决定是直接在当前的io线程decode消息体,还是传给给后面的 DecodeHandler ,内部调用Serialization(默认为hessian2)以完成decode消息体."decode.in.io"默认为false,即交给后面的DecodeHandler处理.
而DecodeHandler则是被AllChannelHandler(默认)装饰,因此会派到线程池里去做消息体decode.<br>

下面重点介绍 Transport 层和 Exchange 层对 Codec2 接口的实现,涉及的类如下图所示:
[![yJUzkT.png](https://s3.ax1x.com/2021/02/05/yJUzkT.png)](https://imgchr.com/i/yJUzkT)

AbstractCodec抽象类并没有实现 Codec2 中定义的接口方法,而是提供了几个给子类用的基础方法,下面简单说明这些方法的功能
* getSerialization() 方法:通过 SPI 获取当前使用的序列化方式
* checkPayload() 方法:检查编解码数据的长度,如果数据超长,会抛出异常
* isClientSide()、isServerSide() 方法:判断当前是 Client 端还是 Server 端

接下来看TransportCodec,可以看到该类上被标记了 @Deprecated 注解,表示已经废弃.TransportCodec 的实现非常简单,其中根据 getSerialization() 方法选择的序列化方法对传入消息或 ChannelBuffer 进行序列化或反序列化,这里就不再介绍 TransportCodec 实现了.<br>

TelnetCodec继承了 TransportCodec 序列化和反序列化的基本能力,同时还提供了对 Telnet 命令处理的能力.<br>

最后来看ExchangeCodec,它在 TelnetCodec 的基础之上,添加了处理协议头的能力.下面是 Dubbo 协议的格式,能够清晰地看出协议中各个数据所占的位数:
[![yJd9VP.png](https://s3.ax1x.com/2021/02/05/yJd9VP.png)](https://imgchr.com/i/yJd9VP)

结合上图,我们来深入了解一下 Dubbo 协议中各个部分的含义:
* 0~7 位和 8~15 位分别是 Magic High 和 Magic Low,是固定魔数值(0xdabb),我们可以通过这两个 Byte,快速判断一个数据包是否为 Dubbo 协议,这也类似 Java 字节码文件里的魔数
* 16 位是 Req/Res 标识,用于标识当前消息是请求还是响应
* 17 位是 2Way 标识,用于标识当前消息是单向还是双向
* 18 位是 Event 标识,用于标识当前消息是否为事件消息
* 19~23 位是序列化类型的标志,用于标识当前消息使用哪一种序列化算法
* 24~31 位是 Status 状态,用于记录响应的状态,仅在 Req/Res 为 0(响应)时有用
* 32~95 位是 Request ID,用于记录请求的唯一标识,类型为 long
* 96~127 位是序列化后的内容长度,该值是按字节计数，int 类型
* 128 位之后是可变的数据,被特定的序列化算法(由序列化类型标志确定)序列化后,每个部分都是一个 byte [] 或者 byte.如果是请求包(Req/Res = 1),则每个部分依次为:Dubbo version、Service name、Service version、Method name、Method parameter types、Method arguments 和 Attachments.如果是响应包(Req/Res = 0),则每个部分依次为:①返回值类型(byte),标识从服务器端返回的值类型,包括返回空值(RESPONSE_NULL_VALUE 2)、正常响应值(RESPONSE_VALUE 1)和异常(RESPONSE_WITH_EXCEPTION 0)三种;②返回值,从服务端返回的响应 bytes.<br>

可以看到 Dubbo 协议中前 128 位是协议头,之后的内容是具体的负载数据.协议头就是通过 ExchangeCodec 实现编解码的.<br>

ExchangeCodec 的核心字段有如下几个:
* HEADER_LENGTH(int 类型,值为 16):协议头的字节数,16 字节,即 128 位
* MAGIC(short 类型,值为 0xdabb):协议头的前 16 位,分为 MAGIC_HIGH 和 MAGIC_LOW 两个字节
* FLAG_REQUEST(byte 类型,值为 0x80):用于设置 Req/Res 标志位
* FLAG_TWOWAY(byte 类型,值为 0x40):用于设置 2Way 标志位
* FLAG_EVENT(byte 类型,值为 0x20):用于设置 Event 标志位
* SERIALIZATION_MASK(int 类型,值为 0x1f):用于获取序列化类型的标志位的掩码

在 ExchangeCodec 的 encode() 方法中会根据需要编码的消息类型进行分类,其中 encodeRequest() 方法专门对 Request 对象进行编码,具体实现如下:
```
protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {

    Serialization serialization = getSerialization(channel);
    byte[] header = new byte[HEADER_LENGTH]; // 该数组用来暂存协议头
    // 在header数组的前两个字节中写入魔数
    Bytes.short2bytes(MAGIC, header);
    // 根据当前使用的序列化设置协议头中的序列化标志位
    header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
    if (req.isTwoWay()) { // 设置协议头中的2Way标志位
        header[2] |= FLAG_TWOWAY;
    }
    if (req.isEvent()) { // 设置协议头中的Event标志位
        header[2] |= FLAG_EVENT;
    }
    // 将请求ID记录到请求头中
    Bytes.long2bytes(req.getId(), header, 4);
    // 下面开始序列化请求，并统计序列化后的字节数
    // 首先使用savedWriteIndex记录ChannelBuffer当前的写入位置
    int savedWriteIndex = buffer.writerIndex();
    // 将写入位置后移16字节
    buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
    // 根据选定的序列化方式对请求进行序列化
    ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
    ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
    if (req.isEvent()) { // 对事件进行序列化
        encodeEventData(channel, out, req.getData());
    } else { // 对Dubbo请求进行序列化，具体在DubboCodec中实现
        encodeRequestData(channel, out, req.getData(), req.getVersion());
    }
    out.flushBuffer();
    if (out instanceof Cleanable) {
        ((Cleanable) out).cleanup();
    }
    bos.flush();
    bos.close(); // 完成序列化
    int len = bos.writtenBytes(); // 统计请求序列化之后，得到的字节数
    checkPayload(channel, len); // 限制一下请求的字节长度
    Bytes.int2bytes(len, header, 12); // 将字节数写入header数组中
    // 下面调整ChannelBuffer当前的写入位置，并将协议头写入Buffer中
    buffer.writerIndex(savedWriteIndex);
    buffer.writeBytes(header); 
    // 最后，将ChannelBuffer的写入位置移动到正确的位置
    buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
}
```

encodeResponse() 方法编码响应的方式与 encodeRequest() 方法编码请求的方式类似,这里就不再展开介绍了,可以翻源码查看.对于既不是 Request,也不是 Response 的消息,ExchangeCodec 会使用从父类继承下来的能力来编码,例如对 telnet 命令的编码.<br>

ExchangeCodec 的 decode() 方法是 encode() 方法的逆过程,会先检查魔数,然后读取协议头和后续消息的长度,最后根据协议头中的各个标志位构造相应的对象,以及反序列化数据.在了解协议头结构的前提下,再去阅读这段逻辑就十分轻松了.<br>



<br><br>
## <font size="3">九. 一点总结</font>

Exchange层抽象出了Request和Response语义,因此这层的逻辑是通过这两个语义实现的.
[![yrmQKA.png](https://s3.ax1x.com/2021/02/12/yrmQKA.png)](https://imgchr.com/i/yrmQKA)