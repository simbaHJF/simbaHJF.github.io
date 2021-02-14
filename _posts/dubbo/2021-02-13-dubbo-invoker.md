---
layout:     post
title:      "Dubbo Invoker"
date:       2021-02-13 18:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---



## 导航
[一. Invoker 前言](#jump1)
<br>
[二. 深入 Invoker](#jump2)
<br>
[三. RpcContext](#jump3)
<br>
[四. DubboInvoker](#jump4)



<br><br>
## <span id="jump1">一. Invoker 前言</span>

**<font size="3">服务提供方来讲</font>** <br>

上层业务 Bean 会被封装成 Invoker 对象,然后传入 DubboProtocol.export() 方法中,该 Invoker 被封装成 DubboExporter,并保存到 exporterMap 集合中缓存.<br>

在 DubboProtocol 暴露的 ProtocolServer 收到请求时,经过一系列解码处理,最终会到达 DubboProtocol.requestHandler 这个 ExchangeHandler 对象中,该 ExchangeHandler 对象会从 exporterMap 集合中取出请求的 Invoker,并调用其 invoke() 方法处理请求<br>


**<font size="3">服务调用方来讲</font>** <br>

DubboProtocol.protocolBindingRefer() 方法则会将底层的 ExchangeClient 集合封装成 DubboInvoker,然后由上层逻辑封装成代理对象,这样业务层就可以像调用本地 Bean 一样,完成远程调用<br>



<br><br>
## <span id="jump2">二. 深入 Invoker</span>

首先，我们来看 AbstractInvoker 这个抽象类,它继承了 Invoker 接口,继承关系如下图所示:
[![ysiSk4.png](https://s3.ax1x.com/2021/02/13/ysiSk4.png)](https://imgchr.com/i/ysiSk4)

最核心的 DubboInvoker 继承自AbstractInvoker 抽象类,AbstractInvoker 的核心字段有如下几个
* type(Class<T> 类型): 该 Invoker 对象封装的业务接口类型,例如 Demo 示例中的 DemoService 接口
* url(URL 类型): 与当前 Invoker 关联的 URL 对象,其中包含了全部的配置信息
* attachment(Map<String, Object> 类型): 当前 Invoker 关联的一些附加信息,这些附加信息可以来自关联的 URL.在 AbstractInvoker 的构造函数的某个重载中,会调用 convertAttachment() 方法,其中就会从关联的 URL 对象获取指定的 KV 值记录到 attachment 集合中
* available(volatile boolean类型)、destroyed(AtomicBoolean 类型): 这两个字段用来控制当前 Invoker 的状态.available 默认值为 true,destroyed 默认值为 false.在 destroy() 方法中会将 available 设置为 false,将 destroyed 字段设置为 true


在 AbstractInvoker 中实现了 Invoker 接口中的 invoke() 方法,这里有点模板方法模式的感觉,其中先对 URL 中的配置信息以及 RpcContext 中携带的附加信息进行处理,添加到 Invocation 中作为附加信息,然后调用 doInvoke() 方法发起调用(该方法由 AbstractInvoker 的子类具体实现),最后得到 AsyncRpcResult 对象返回
```
public Result invoke(Invocation inv) throws RpcException {

    // 首先将传入的Invocation转换为RpcInvocation
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    // 将attachment集合添加为Invocation的附加信息
    if (CollectionUtils.isNotEmptyMap(attachment)) {
        invocation.addObjectAttachmentsIfAbsent(attachment);
    }
    // 将RpcContext的附加信息添加为Invocation的附加信息
    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
        invocation.addObjectAttachments(contextAttachments);
    }
    // 设置此次调用的模式，异步还是同步
    invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
    // 如果是异步调用，给这次调用添加一个唯一ID
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    AsyncRpcResult asyncResult;
    try { // 调用子类实现的doInvoke()方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
    } catch (InvocationTargetException e) {// 省略异常处理的逻辑
    } catch (RpcException e) { // 省略异常处理的逻辑
    } catch (Throwable e) {
        asyncResult = AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
    }
    RpcContext.getContext().setFuture(new FutureAdapter(asyncResult.getResponseFuture()));
    return asyncResult;
}
```

接下来,需要深入介绍的第一个类是 RpcContext.<br>



<br><br>
## <span id="jump3">三. RpcContext</span>

RpcContext 是线程级别的上下文信息,每个线程绑定一个 RpcContext 对象,底层依赖 ThreadLocal 实现,RpcContext 主要用于存储一个线程中一次请求的临时状态,当线程处理新的请求(Provider 端)或是线程发起新的请求(Consumer 端)时,RpcContext 中存储的内容就会更新.<br>

下面来看 RpcContext 中两个InternalThreadLocal的核心字段,这两个字段的定义如下所示
```
// 在发起请求时，会使用该RpcContext来存储上下文信息
private static final InternalThreadLocal<RpcContext> LOCAL = new InternalThreadLocal<RpcContext>() {
    @Override
    protected RpcContext initialValue() {
        return new RpcContext();
    }
};

// 在接收到响应的时候，会使用该RpcContext来存储上下文信息
private static final InternalThreadLocal<RpcContext> SERVER_LOCAL = ...
```

InternalThreadLocal是Dubbo内部自己实现的一套ThreadLocal,与JDK中提供的类似,但有细微差别,这里不再展开分析原理,可翻看源码查阅.<br>

RpcContext 作为调用的上下文信息,可以记录非常多的信息,下面介绍其中的一些核心字段
* attachments(Map<String, Object> 类型): 可用于记录调用上下文的附加信息,这些信息会被添加到 Invocation 中,并传递到远端节点
* values(Map<String, Object> 类型): 用来记录上下文的键值对信息,但是不会被传递到远端节点
* methodName、parameterTypes、arguments: 分别用来记录调用的方法名、参数类型列表以及具体的参数列表,与相关 Invocation 对象中的信息一致
* localAddress、remoteAddress(InetSocketAddress 类型): 记录了自己和远端的地址
* request、response(Object 类型): 可用于记录底层关联的请求和响应
* asyncContext(AsyncContext 类型): 异步Context, 其中可以存储异步调用相关的 RpcContext 以及异步请求相关的 Future



<br><br>
## <span id="jump4">四. DubboInvoker</span>

通过前面对 DubboProtocol 的分析我们知道,protocolBindingRefer() 方法会根据调用的业务接口类型以及 URL 创建底层的 ExchangeClient 集合,然后封装成 DubboInvoker 对象返回.DubboInvoker 是 AbstractInvoker 的实现类,在其 doInvoke() 方法中首先会选择此次调用使用 ExchangeClient 对象,然后确定此次调用是否需要返回值,最后调用 ExchangeClient.request() 方法发送请求,对返回的 Future 进行简单封装并返回
```
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    // 此次调用的方法名称
    final String methodName = RpcUtils.getMethodName(invocation);
    // 向Invocation中添加附加信息，这里将URL的path和version添加到附加信息中
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);
    ExchangeClient currentClient; // 选择一个ExchangeClient实例
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
    // 根据调用的方法名称和配置计算此次调用的超时时间
    int timeout = calculateTimeout(invocation, methodName); 
    if (isOneway) { // 不需要关注返回值的请求
        boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
        currentClient.send(inv, isSent);
        return AsyncRpcResult.newDefaultAsyncResult(invocation);
    } else { // 需要关注返回值的请求
        // 获取处理响应的线程池，对于同步请求，会使用ThreadlessExecutor，ThreadlessExecutor的原理前面已经分析过了，这里不再赘述；对于异步请求，则会使用共享的线程池，ExecutorRepository接口的相关设计和实现在前面已经详细分析过了，这里不再重复。
        ExecutorService executor = getCallbackExecutor(getUrl(), inv);
        // 使用上面选出的ExchangeClient执行request()方法，将请求发送出去
        CompletableFuture<AppResponse> appResponseFuture =
                currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
        // 这里将AppResponse封装成AsyncRpcResult返回
        AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
        result.setExecutor(executor);
        return result;
    }
}
```

在 DubboInvoker.invoke() 方法中有一些细节需要关注一下.首先是根据 URL 以及 Invocation 中的配置,决定此次调用是否为oneway 调用方式.
```
public static boolean isOneway(URL url, Invocation inv) {
    boolean isOneway;
    if (Boolean.FALSE.toString().equals(inv.getAttachment(RETURN_KEY))) {
        isOneway = true; // 首先关注的是Invocation中"return"这个附加属性
    } else {
        isOneway = !url.getMethodParameter(getMethodName(inv), RETURN_KEY, true); // 之后关注URL中，调用方法对应的"return"配置
    }
    return isOneway;
}
```
若为oneway方式,在发送完oneway 请求之后,会立即创建一个已完成状态的 AsyncRpcResult 对象（主要是其中的 responseFuture 是已完成状态）

oneway 指的是客户端发送消息后,不需要得到响应.所以,对于那些不关心服务端响应的请求,就比较适合使用 oneway 通信,如下图所示
[![ysGU5d.png](https://s3.ax1x.com/2021/02/13/ysGU5d.png)](https://imgchr.com/i/ysGU5d)

可以看到发送 oneway 请求的方式是send() 方法,而后面发送 twoway 请求的方式是 request() 方法.通过之前的分析我们知道,request() 方法会相应地创建 DefaultFuture 对象以及检测超时的定时任务,而 send() 方法则不会创建这些东西,它是直接将 Invocation 包装成 oneway 类型的 Request 发送出去.<br>

在服务端的 HeaderExchangeHandler.receive() 方法中,会针对 oneway 请求和 twoway 请求执行不同的分支处理:twoway 请求由 handleRequest() 方法进行处理,其中会关注调用结果并形成 Response 返回给客户端;oneway 请求则直接交给上层的 DubboProtocol.requestHandler,完成方法调用之后,不会返回任何 Response.<br>

我们就结合如下示例代码来简单说明一下 HeaderExchangeHandler.request() 方法中的相关片段
```
public void received(Channel channel, Object message) throws RemotingException {
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    if (message instanceof Request) {
        if (request.isTwoWay()) {
            handleRequest(exchangeChannel, request);
        } else {
            handler.received(exchangeChannel, request.getData());
        }
    } else ... // 省略其他分支的展示
}
```

DubboInvoker在发送完oneway 请求之后,会立即创建一个已完成状态的 AsyncRpcResult 对象(主要是其中的 responseFuture 是已完成状态).下面再来看下DubboInvoker处理twoway请求和响应的相关实现,其中会涉及响应解码、同步/异步响应等相关内容.<br>

那么DubboInvoker 对twoway 请求的处理又是怎样的呢?首先,DubboInvoker 会调用 getCallbackExecutor() 方法,根据不同的 InvokeMode 返回不同的线程池实现,代码如下:
```
protected ExecutorService getCallbackExecutor(URL url, Invocation inv) {
    ExecutorService sharedExecutor = ExtensionLoader.getExtensionLoader(ExecutorRepository.class).getDefaultExtension().getExecutor(url);
    if (InvokeMode.SYNC == RpcUtils.getInvokeMode(getUrl(), inv)) {
        return new ThreadlessExecutor(sharedExecutor);
    } else {
        return sharedExecutor;
    }
}
```

InvokeMode 有三个可选值,分别是 SYNC、ASYNC 和 FUTURE.这里对于 SYNC 模式返回的线程池是 ThreadlessExecutor,至于其他两种异步模式,会根据 URL 选择对应的共享线程池.<br>

SYNC 表示同步模式,是 Dubbo 的默认调用模式,具体含义如下图所示,客户端发送请求之后,客户端线程会阻塞等待服务端返回响应.
[![ysYTN4.png](https://s3.ax1x.com/2021/02/13/ysYTN4.png)](https://imgchr.com/i/ysYTN4)

在拿到线程池之后,DubboInvoker 就会调用 ExchangeClient.request() 方法,将 Invocation 包装成 Request 请求发送出去,同时会创建相应的 DefaultFuture 返回.注意,这里还加了一个回调,取出其中的 AppResponse 对象.AppResponse 表示的是服务端返回的具体响应,
* result(Object 类型): 响应结果,也就是服务端返回的结果值
* exception(Throwable 类型): 服务端返回的异常信息
* attachments(Map<String, Object> 类型): 服务端返回的附加信息

这里请求返回的 AppResponse 你可能不太熟悉,但是其子类 DecodeableRpcResult 你可能就有点眼熟了,DecodeableRpcResult 表示的是一个响应,与其对应的是 DecodeableRpcInvocation(它表示的是请求)

**<font size="3">DecodeableRpcResult</font>** <br>

DecodeableRpcResult 解码核心流程大致如下:
* 首先,确定当前使用的序列化方式,并对字节流进行解码
* 然后,读取一个 byte 的标志位,其可选值有六种枚举,下面我们就以其中的 RESPONSE_VALUE_WITH_ATTACHMENTS 为例进行分析
* 标志位为 RESPONSE_VALUE_WITH_ATTACHMENTS 时,会先通过 handleValue() 方法处理返回值,其中会根据 RpcInvocation 中记录的返回值类型读取返回值,并设置到 result 字段
* 最后,再通过 handleAttachment() 方法读取返回的附加信息,并设置到 DecodeableRpcResult 的 attachments 字段中

```
public Object decode(Channel channel, InputStream input) throws IOException {
    // 反序列化
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
            .deserialize(channel.getUrl(), input);
    byte flag = in.readByte(); // 读取一个byte的标志位
    // 根据标志位判断当前结果中包含的信息，并调用不同的方法进行处理
    switch (flag) { 
        case DubboCodec.RESPONSE_NULL_VALUE:
            break;
        case DubboCodec.RESPONSE_VALUE:
            handleValue(in);
            break;
        case DubboCodec.RESPONSE_WITH_EXCEPTION:
            handleException(in);
            break;
        case DubboCodec.RESPONSE_NULL_VALUE_WITH_ATTACHMENTS:
            handleAttachment(in);
            break;
        case DubboCodec.RESPONSE_VALUE_WITH_ATTACHMENTS:
            handleValue(in);
            handleAttachment(in);
            break;
        case DubboCodec.RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS:
        default:
            throw new IOException("..." );
    }
    if (in instanceof Cleanable) {
        ((Cleanable) in).cleanup();
    }
    return this;
}
```

**<font size="3">AsyncRpcResult</font>** <br>

在 DubboInvoker 中有一个 AsyncRpcResult 类,它表示的是一个异步的、未完成的 RPC 调用,其中会记录对应 RPC 调用的信息(例如,关联的 RpcContext 和 Invocation 对象),包括以下几个核心字段
* responseFuture(CompletableFuture<AppResponse> 类型): 这个 responseFuture 字段与前文提到的 DefaultFuture 有紧密的联系,是 DefaultFuture 回调链上的一个 Future.后面 AsyncRpcResult 之上添加的回调,实际上都是添加到这个 Future 之上
* storedContext、storedServerContext(RpcContext 类型): 用于存储相关的 RpcContext 对象.我们知道 RpcContext 是与线程绑定的,而真正执行 AsyncRpcResult 上添加的回调方法的线程可能先后处理过多个不同的 AsyncRpcResult,所以我们需要传递并保存当前的 RpcContext
* executor(Executor 类型): 此次 RPC 调用关联的线程池
* invocation(Invocation 类型): 此次 RPC 调用关联的 Invocation 对象

AsyncRpcResult、AppResponse、DecodeableRpcResult三者关系如下图所示:
[![yySJOO.png](https://s3.ax1x.com/2021/02/14/yySJOO.png)](https://imgchr.com/i/yySJOO)

DecodeableRpcResult是AppResponse的子类,AsyncRpcResult中通过responseFuture字段持有CompletableFuture<AppResponse>.<br>

在前面的分析中我们看到,RpcInvocation.InvokeMode 字段中可以指定调用为 SYNC 模式,也就是同步调用模式,那 AsyncRpcResult 这种异步设计是如何支持同步调用的呢? 在 AbstractProtocol.refer() 方法中,Dubbo 会将 DubboProtocol.protocolBindingRefer() 方法返回的 Invoker 对象(即 DubboInvoker 对象)用 AsyncToSyncInvoker 封装一层.
```
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

AsyncToSyncInvoker 是 Invoker 的装饰器,负责将异步调用转换成同步调用,其 invoke() 方法的核心实现如下
```
public Result invoke(Invocation invocation) throws RpcException {
    Result asyncResult = invoker.invoke(invocation);
    if (InvokeMode.SYNC == ((RpcInvocation) invocation).getInvokeMode()) {
        // 调用get()方法，阻塞等待响应返回
        asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
    }
    return asyncResult;
}
```

其实 AsyncRpcResult.get() 方法底层调用的就是 responseFuture 字段的 get() 方法,对于同步请求来说,会先调用 ThreadlessExecutor.waitAndDrain() 方法阻塞等待响应返回,具体实现如下所示:
```
public Result get() throws InterruptedException, ExecutionException {
    if (executor != null && executor instanceof ThreadlessExecutor) {
        // 针对ThreadlessExecutor的特殊处理，这里调用waitAndDrain()等待响应
        ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
        threadlessExecutor.waitAndDrain();
    }
    // 非ThreadlessExecutor线程池的场景中，则直接调用Future(最底层是DefaultFuture)的get()方法阻塞
    return responseFuture.get();
}
```

AsyncToSyncInvoker的invoke方法,体现了Dubbo的同步和异步调用的逻辑.Dubbo 实现同步和异步调用比较关键的一点就在于由谁调用 ResponseFuture 的 get 方法.同步调用模式下,由框架自身调用 ResponseFuture 的 get 方法.异步调用模式下,则由用户调用该方法.<br>

DubboInvoker 涉及的同步调用、异步调用的原理和底层实现就介绍到这里了,我们可以通过一张流程图进行简单总结,如下所示
[![yySjhR.png](https://s3.ax1x.com/2021/02/14/yySjhR.png)](https://imgchr.com/i/yySjhR)

在 Client 端发送请求时,首先会创建对应的 DefaultFuture(其中记录了请求 ID 等信息),然后依赖 Netty 的异步发送特性将请求发送到 Server 端.需要说明的是,这整个发送过程是不会阻塞任何线程的.之后,将 DefaultFuture 返回给上层,在这个返回过程中,DefaultFuture 会被封装成 AsyncRpcResult,同时也可以添加回调函数.<br>

当 Client 端接收到响应结果的时候,会交给关联的线程池(ExecutorService)或是业务线程(使用 ThreadlessExecutor 场景)进行处理,得到 Server 返回的真正结果.拿到真正的返回结果后,会将其设置到 DefaultFuture 中,并调用 complete() 方法将其设置为完成状态.此时,就会触发前面注册在 DefaulFuture 上的回调函数,执行回调逻辑.<br>

