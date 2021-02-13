---
layout:     post
title:      "Dubbo RPC Protocol层服务暴露流程"
date:       2021-02-06 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---



## 导航
[一. Protocol 接口的继承关系](#jump1)
<br>
[二. export 流程](#jump2)
<br>
[三. refer 流程](#jump3)
<br>
[四. destroy方法](#jump4)






<br><br>
## <span id="jump1">一. Protocol 接口的继承关系</span>

先来看下 Protocol 接口的继承关系
[![yY6Cz6.png](https://s3.ax1x.com/2021/02/06/yY6Cz6.png)](https://imgchr.com/i/yY6Cz6)

其中,AbstractProtocol提供了一些 Protocol 实现需要的公共能力以及公共字段,它的核心字段有如下三个
* exporterMap(Map<String, Exporter<?>\>类型): 用于存储暴露的服务集合,其中的 Key 通过 ProtocolUtils.serviceKey() 方法创建的服务标识,在 ProtocolUtils 中维护了多层的 Map 结构(如下图所示).首先按照 group 分组,在实践中我们可以根据需求设置 group,例如,按照机房、地域等进行 group 划分,做到就近调用;在 GroupServiceKeyCache 中,依次按照 serviceName、serviceVersion、port 进行分类,最终缓存的 serviceKey 是前面三者拼接而成的
[![yY6sw4.png](https://s3.ax1x.com/2021/02/06/yY6sw4.png)](https://imgchr.com/i/yY6sw4)
* serverMap(Map<String, ProtocolServer>类型):记录了全部的 ProtocolServer 实例,其中的 Key 是 host 和 port 组成的字符串,Value 是监听该地址的 ProtocolServer. ProtocolServer 就是对 RemotingServer 的一层简单封装,表示一个服务端.
* invokers(Set<Invoker<?>>类型): 服务引用的集合

AbstractProtocol 没有对 Protocol 的 export() 方法进行实现,对 refer() 方法的实现也是委托给了 protocolBindingRefer() 这个抽象方法,然后由子类实现.AbstractProtocol 唯一实现的方法就是 destory() 方法,其首先会遍历 Invokers 集合,销毁全部的服务引用,然后遍历全部的 exporterMap 集合,销毁发布出去的服务,具体实现如下
```
public void destroy() {

    for (Invoker<?> invoker : invokers) {
        if (invoker != null) {
            invokers.remove(invoker);
            invoker.destroy(); // 关闭全部的服务引用
        }
    }
    for (String key : new ArrayList<String>(exporterMap.keySet())) {
        Exporter<?> exporter = exporterMap.remove(key);
        if (exporter != null) {
            exporter.unexport(); // 关闭暴露出去的服务
        }
    }
}
```


<br><br>
## <span id="jump2">二. export 流程</span>

了解了 AbstractProtocol 提供的公共能力之后,我们再来分析Dubbo 默认使用的 Protocol 实现类—— DubboProtocol 实现.这里我们首先关注 DubboProtocol 的 export() 方法,也就是服务发布的相关实现,如下所示:
```
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // 创建ServiceKey，其核心实现在前文已经详细分析过了，这里不再重复
    String key = serviceKey(url);

    // 将上层传入的Invoker对象封装成DubboExporter对象，然后记录到exporterMap集合中
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);
    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        }
    }
    // 启动ProtocolServer
    openServer(url);

    // 进行序列化的优化处理
    optimizeSerialization(url);
    return exporter;
}
```


<br>
**<font size="3">1. DubboExporter</font>** <br>

这里涉及的第一个点是 DubboExporter 对 Invoker 的封装,DubboExporter 的继承关系如下图所示:
[![yY2btJ.png](https://s3.ax1x.com/2021/02/06/yY2btJ.png)](https://imgchr.com/i/yY2btJ)

AbstractExporter 中维护了一个 Invoker 对象,以及一个 unexported 字段(boolean 类型),在 unexport() 方法中会设置 unexported 字段为 true,并调用 Invoker 对象的 destory() 方法进行销毁.<br>

DubboExporter 也比较简单,其中会维护底层 Invoker 对应的 ServiceKey 以及 DubboProtocol 中的 exporterMap 集合,在其 unexport() 方法中除了会调用父类 AbstractExporter 的 unexport() 方法之外,还会清理该 DubboExporter 实例在 exporterMap 中相应的元素<br>


<br>
**<font size="3">2. 服务端初始化</font>** <br>

了解了 Exporter 实现之后,我们继续看 DubboProtocol 中服务发布的流程.从下面这张调用关系图中可以看出,openServer() 方法会一路调用前面介绍的 Exchange 层、Transport 层,并最终创建 NettyServer 来接收客户端的请求
[![yYRcDK.png](https://s3.ax1x.com/2021/02/06/yYRcDK.png)](https://imgchr.com/i/yYRcDK)

下面我们将逐个介绍 export() 方法栈中的每个被调用的方法.<br>

首先,在 openServer() 方法中会根据 URL 判断当前是否为服务端,只有服务端才能创建 ProtocolServer 并对外服务.如果是来自服务端的调用,会依靠 serverMap 集合检查是否已有 ProtocolServer 在监听 URL 指定的地址;如果没有,会调用 createServer() 方法进行创建.openServer() 方法的具体实现如下
```
private void openServer(URL url) {

    String key = url.getAddress(); // 获取host:port这个地址
    boolean isServer = url.getParameter(IS_SERVER_KEY, true);
    if (isServer) { // 只有Server端才能启动Server对象
        ProtocolServer server = serverMap.get(key);
        if (server == null) { // 无ProtocolServer监听该地址
            synchronized (this) { // DoubleCheck，防止并发问题
                server = serverMap.get(key);
                if (server == null) {
                    // 调用createServer()方法创建ProtocolServer对象
                    serverMap.put(key, createServer(url));
                }
            }
        } else { 
            // 如果已有ProtocolServer实例，则尝试根据URL信息重置ProtocolServer
            server.reset(url);
        }
    }
}
```

createServer() 方法首先会为 URL 添加一些默认值,同时会进行一些参数值的检测,主要有五个:
* HEARTBEAT_KEY 参数值,默认值为 60000,表示默认的心跳时间间隔为 60 秒
* CHANNEL_READONLYEVENT_SENT_KEY 参数值,默认值为 true,表示 ReadOnly 请求需要阻塞等待响应返回.在 Server 关闭的时候,只能发送 ReadOnly 请求,这些 ReadOnly 请求由这里设置的 CHANNEL_READONLYEVENT_SENT_KEY 参数值决定是否需要等待响应返回
* CODEC_KEY 参数值,默认值为 dubbo.你可以回顾 Codec2 接口中 @Adaptive 注解的参数,都是获取该 URL 中的 CODEC_KEY 参数值
* 检测 SERVER_KEY 参数指定的扩展实现名称是否合法,默认值为 netty.你可以回顾 Transporter 接口中 @Adaptive 注解的参数,它决定了 Transport 层使用的网络库实现,默认使用 Netty 4 实现
* 检测 CLIENT_KEY 参数指定的扩展实现名称是否合法.同 SERVER_KEY 参数的检查流程

完成上述默认参数值的设置之后,就可以通过 Exchangers 门面类创建 ExchangeServer,并封装成 DubboProtocolServer 返回
```
private ProtocolServer createServer(URL url) {
    url = URLBuilder.from(url)
            // send readonly event when server closes, it's enabled by default
            // ReadOnly请求是否阻塞等待
            .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
            // enable heartbeat by default
            // 心跳间隔
            .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
            // Codec2扩展实现
            .addParameter(CODEC_KEY, DubboCodec.NAME)
            .build();
	// 检测SERVER_KEY参数指定的Transporter扩展实现是否合法
    String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);
    }
    ExchangeServer server;
    try {
    	// 通过Exchangers门面类，创建ExchangeServer对象
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    // 将ExchangeServer封装成DubboProtocolServer返回
    return new DubboProtocolServer(server);
}
```
在 createServer() 方法中还有几个细节需要展开分析一下.第一个是创建 ExchangeServer 时,使用的 Codec2 接口实现实际上是 DubboCountCodec.<br>

DubboCountCodec 中维护了一个 DubboCodec 对象,编解码的能力都是 DubboCodec 提供的,DubboCountCodec 只负责在解码过程中 ChannelBuffer 的 readerIndex 指针控制,具体实现如下
```
public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {

    int save = buffer.readerIndex(); // 首先保存readerIndex指针位置
    // 创建MultiMessage对象，其中可以存储多条消息
    MultiMessage result = MultiMessage.create(); 
    do {
        // 通过DubboCodec提供的解码能力解码一条消息
        Object obj = codec.decode(channel, buffer);
        // 如果可读字节数不足一条消息，则会重置readerIndex指针
        if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
            buffer.readerIndex(save);
            break;
        } else { // 将成功解码的消息添加到MultiMessage中暂存
            result.addMessage(obj);
            logMessageLength(obj, buffer.readerIndex() - save);
            save = buffer.readerIndex();
        }
    } while (true);
    if (result.isEmpty()) { // 一条消息也未解码出来，则返回NEED_MORE_INPUT错误码
        return Codec2.DecodeResult.NEED_MORE_INPUT;
    }
    if (result.size() == 1) { // 只解码出来一条消息，则直接返回该条消息
        return result.get(0);
    }
    // 解码出多条消息的话，会将MultiMessage返回
    return result;
}
```

DubboCountCodec、DubboCodec 都实现了 Codec2 接口,其中 DubboCodec 是 ExchangeCodec 的子类
[![yY5MSH.png](https://s3.ax1x.com/2021/02/06/yY5MSH.png)](https://imgchr.com/i/yY5MSH)

ExchangeCodec 只处理了 Dubbo 协议的请求头,而 DubboCodec 则是通过继承的方式,在 ExchangeCodec 基础之上,添加了解析 Dubbo 消息体的功能. ExchangeCodec 中的 encodeRequest() 方法完成对 Request 请求的编码, 其中会调用 encodeRequestData() 方法完成请求体的编码. DubboCodec 中就覆盖了 encodeRequestData() 方法,按照 Dubbo 协议的格式编码 Request 请求体,具体实现如下:
```
protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {

    // 请求体相关的内容，都封装在了RpcInvocation
    RpcInvocation inv = (RpcInvocation) data; 
    out.writeUTF(version); // 写入版本号
    String serviceName = inv.getAttachment(INTERFACE_KEY);
    if (serviceName == null) {
        serviceName = inv.getAttachment(PATH_KEY);
    }
    // 写入服务名称
    out.writeUTF(serviceName);
    // 写入Service版本号
    out.writeUTF(inv.getAttachment(VERSION_KEY));
    // 写入方法名称
    out.writeUTF(inv.getMethodName());
    // 写入参数类型列表
    out.writeUTF(inv.getParameterTypesDesc());
    // 依次写入全部参数
    Object[] args = inv.getArguments();
    if (args != null) {
        for (int i = 0; i < args.length; i++) {
            out.writeObject(encodeInvocationArgument(channel, inv, i));
        }
    }
    // 依次写入全部的附加信息
    out.writeAttachments(inv.getObjectAttachments());
}
```

RpcInvocation 实现了 Invocation 接口,如下图所示
[![yYoCrR.png](https://s3.ax1x.com/2021/02/06/yYoCrR.png)](https://imgchr.com/i/yYoCrR)

下面是 RpcInvocation 中的核心字段,通过读写这些字段即可实现 Invocation 接口的全部方法:
* targetServiceUniqueName(String类型): 要调用的唯一服务名称,其实就是 ServiceKey,即 group/interface:version 三部分构成的字符串
* methodName(String类型): 调用的目标方法名称
* serviceName(String类型): 调用的目标服务名称,示例中就是org.apache.dubbo.demo.DemoService
* parameterTypes(Class<?>[]类型): 记录了目标方法的全部参数类型
* parameterTypesDesc(String类型): 参数列表签名
* arguments(Object[]类型): 具体参数值
* attachments(Map<String, Object>类型): 此次调用的附加信息,可以被序列化到请求中
* attributes(Map<Object, Object>类型): 此次调用的属性信息,这些信息不能被发送出去
* invoker(Invoker<?>类型): 此次调用关联的 Invoker 对象
* returnType(Class<?>类型): 返回值的类型
* invokeMode(InvokeMode类型): 此次调用的模式,分为 SYNC、ASYNC 和 FUTURE 三类


在上面的继承图中看到 RpcInvocation 的一个子类----DecodeableRpcInvocation,它是用来支持解码的,其实现的 decode() 方法正好是 DubboCodec.encodeRequestData() 方法对应的解码操作,在 DubboCodec.decodeBody() 方法中就调用了这个方法,调用关系如下图所示
[![yYqQ4x.png](https://s3.ax1x.com/2021/02/06/yYqQ4x.png)](https://imgchr.com/i/yYqQ4x)

这个解码过程中有个细节,在 DubboCodec.decodeBody() 方法中有如下代码片段,其中会根据 DECODE_IN_IO_THREAD_KEY 这个参数决定是否在 DubboCodec 中进行**<font color="red">消息体解码</font>**(DubboCodec 是在 IO 线程中调用的),对于消息头,肯定是在IO线程中读取的(当然,消息头按字节位进行标记设置,不需要解码,只是按协议格式读取各位代表的信息即可)
```
// decode request.
Request req = new Request(id);
... // 省略Request中其他字段的设置
Object data;
DecodeableRpcInvocation inv;
// 这里会检查DECODE_IN_IO_THREAD_KEY参数
if (channel.getUrl().getParameter(DECODE_IN_IO_THREAD_KEY, DEFAULT_DECODE_IN_IO_THREAD)) {
    inv = new DecodeableRpcInvocation(channel, req, is, proto);
    inv.decode(); // 直接调用decode()方法在当前IO线程中解码
} else { // 这里只是读取数据，不会调用decode()方法在当前IO线程中进行解码
    inv = new DecodeableRpcInvocation(channel, req,
            new UnsafeByteArrayInputStream(readMessageData(is)), proto);
}
data = inv;
req.setData(data); // 设置到Request请求的data字段
return req;
```
如果不在 DubboCodec 中解码,那会在哪里解码呢?回顾 DecodeHandler(Transport 层),它的 received() 方法也是可以进行解码的,另外,DecodeableRpcInvocation 中有一个 hasDecoded 字段来判断当前是否已经完成解码,这样,三者配合就可以根据 DECODE_IN_IO_THREAD_KEY 参数决定执行解码操作的线程了.<br>


这里我们就直接以 AllDispatcher 实现为例给出结论
* IO 线程内执行的 ChannelHandler 实现依次有:InternalEncoder、InternalDecoder(两者底层都是调用 DubboCodec)、IdleStateHandler、MultiMessageHandler、HeartbeatHandler 和 NettyServerHandler
* 在非 IO 线程内执行的 ChannelHandler 实现依次有:DecodeHandler、HeaderExchangeHandler 和 DubboProtocol$requestHandler

在 DubboProtocol 中有一个 requestHandler 字段,它是一个实现了 ExchangeHandlerAdapter 抽象类的匿名内部类的实例,间接实现了 ExchangeHandler 接口,其核心是 reply() 方法,具体实现如下
```
public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

    ... // 这里省略了检查message类型的逻辑，通过前面Handler的处理，这里收到的message必须是Invocation类型的对象
    Invocation inv = (Invocation) message;
    // 获取此次调用Invoker对象
    Invoker<?> invoker = getInvoker(channel, inv);
    ... // 针对客户端回调的内容，在后面详细介绍，这里不再展开分析
    // 将客户端的地址记录到RpcContext中
    RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
    // 执行真正的调用
    Result result = invoker.invoke(inv);
    // 返回结果
    return result.thenApply(Function.identity());
}
```

其中 getInvoker() 方法会先根据 Invocation 携带的信息构造 ServiceKey,然后从 exporterMap 集合中查找对应的 DubboExporter 对象,并从中获取底层的 Invoker 对象返回,具体实现如下
```
Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {

    ... // 省略对客户端Callback以及stub的处理逻辑，后面单独介绍
    String serviceKey = serviceKey(port, path, (String) inv.getObjectAttachments().get(VERSION_KEY),
            (String) inv.getObjectAttachments().get(GROUP_KEY));
    DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
    ... //  查找不到相应的DubboExporter对象时，会直接抛出异常，这里省略了这个检测
    return exporter.getInvoker(); // 获取exporter中获取Invoker对象
}
```

到这里,我们终于见到了对 Invoker 对象的调用,对 Invoker 实现的介绍和分析,在以后会深入介绍这里就先专注于 DubboProtocol 的相关内容


<br>
**<font size="3">3. 序列化优化处理</font>** <br>

下面我们回到 DubboProtocol.export() 方法继续分析,在完成 ProtocolServer 的启动之后,export() 方法最后会调用 optimizeSerialization() 方法对指定的序列化算法进行优化<br>

这里先介绍一个基础知识,在使用某些序列化算法(例如, Kryo、FST 等)时,为了让其能发挥出最佳的性能,最好将那些需要被序列化的类提前注册到 Dubbo 系统中.例如,我们可以通过一个实现了 SerializationOptimizer 接口的优化器,并在配置中指定该优化器,如下示例代码
```
public class SerializationOptimizerImpl implements SerializationOptimizer {

    public Collection<Class> getSerializableClasses() {
        List<Class> classes = new ArrayList<>();
        classes.add(xxxx.class); // 添加需要被序列化的类
        return classes;
    }
}
```

在 DubboProtocol.optimizeSerialization() 方法中,就会获取该优化器中注册的类,通知底层的序列化算法进行优化,序列化的性能将会被大大提升.当然,在进行序列化的时候,难免会级联到很多 Java 内部的类(例如,数组、各种集合类型等),Kryo、FST 等序列化算法已经自动将JDK 中的常用类进行了注册,所以无须重复注册它们.

下面我们回头来看 optimizeSerialization() 方法,分析序列化优化操作的具体实现细节
```
private void optimizeSerialization(URL url) throws RpcException {

    // 根据URL中的optimizer参数值，确定SerializationOptimizer接口的实现类
    String className = url.getParameter(OPTIMIZER_KEY, "");
    Class clazz = Thread.currentThread().getContextClassLoader().loadClass(className);
    // 创建SerializationOptimizer实现类的对象
    SerializationOptimizer optimizer = (SerializationOptimizer) clazz.newInstance();
    // 调用getSerializableClasses()方法获取需要注册的类
    for (Class c : optimizer.getSerializableClasses()) {
        SerializableClassRegistry.registerClass(c); 
    }
    optimizers.add(className);
}
```

SerializableClassRegistry 底层维护了一个 static 的 Map(REGISTRATIONS 字段),registerClass() 方法就是将待优化的类写入该集合中暂存,在使用 Kryo、FST 等序列化算法时,会读取该集合中的类,完成注册操作,相关的调用关系如下图所示
[![yYLgSK.png](https://s3.ax1x.com/2021/02/06/yYLgSK.png)](https://imgchr.com/i/yYLgSK)

按照 Dubbo 官方文档的说法,即使不注册任何类进行优化,Kryo 和 FST 的性能依然普遍优于Hessian2 和 Dubbo 序列化.



<br><br>
## <span id="jump3">三. refer 流程</span>

DubboProtocol 中引用服务的相关实现,其核心实现在 protocolBindingRefer() 方法中:
```
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {

    optimizeSerialization(url); // 进行序列化优化，注册需要优化的类
    // 创建DubboInvoker对象
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    // 将上面创建DubboInvoker对象添加到invoker集合之中
    invokers.add(invoker); 
    return invoker;
}
```

关于 DubboInvoker 的具体实现,我们先暂时不做深入分析.这里我们需要先关注的是getClients() 方法,它创建了底层发送请求和接收响应的 Client 集合,其核心分为了两个部分,一个是针对共享连接的处理,另一个是针对独享连接的处理,具体实现如下:
```
private ExchangeClient[] getClients(URL url) {

    // 是否使用共享连接
    boolean useShareConnect = false;
    // CONNECTIONS_KEY参数值决定了后续建立连接的数量
    int connections = url.getParameter(CONNECTIONS_KEY, 0);
    List<ReferenceCountExchangeClient> shareClients = null;
    if (connections == 0) { // 如果没有连接数的相关配置，默认使用共享连接的方式
        useShareConnect = true;
        // 确定建立共享连接的条数，默认只建立一条共享连接
        String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
        connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
        // 创建公共ExchangeClient集合
        shareClients = getSharedClient(url, connections);
    }
    // 整理要返回的ExchangeClient集合
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (useShareConnect) {
            clients[i] = shareClients.get(i);
        } else {
            // 不使用公共连接的情况下，会创建单独的ExchangeClient实例
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

当使用独享连接的时候,对每个 Service 建立固定数量的 Client,每个 Client 维护一个底层连接.如下图所示,就是针对每个 Service 都启动了两个独享连接:
[![yNR3jI.png](https://s3.ax1x.com/2021/02/07/yNR3jI.png)](https://imgchr.com/i/yNR3jI)


当使用共享连接的时候,会区分不同的网络地址(host:port),一个地址只建立固定数量的共享连接.如下图所示,Provider 1 暴露了多个服务,Consumer 引用了 Provider 1 中的多个服务,共享连接是说 Consumer 调用 Provider 1 中的多个服务时,是通过固定数量的共享 TCP 长连接进行数据传输,这样就可以达到减少服务端连接数的目的.
[![yNRqUO.png](https://s3.ax1x.com/2021/02/07/yNRqUO.png)](https://imgchr.com/i/yNRqUO)


<br>
**<font size="3">创建共享连接</font>** <br>

创建共享连接的实现细节是在 getSharedClient() 方法中,它首先从 referenceClientMap 缓存(Map<String, List<ReferenceCountExchangeClient>> 类型)中查询 Key(host 和 port 拼接成的字符串)对应的共享 Client 集合,如果查找到的 Client 集合全部可用,则直接使用这些缓存的 Client,否则要创建新的 Client 来补充替换缓存中不可用的 Client.示例代码如下:

```
private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {

    String key = url.getAddress(); // 获取对端的地址(host:port)
    // 从referenceClientMap集合中，获取与该地址连接的ReferenceCountExchangeClient集合
    List<ReferenceCountExchangeClient> clients = referenceClientMap.get(key);
    // checkClientCanUse()方法中会检测clients集合中的客户端是否全部可用
    if (checkClientCanUse(clients)) { 
        batchClientRefIncr(clients); // 客户端全部可用时
        return clients;
    }
    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) { // 针对指定地址的客户端进行加锁，分区加锁可以提高并发度
        clients = referenceClientMap.get(key);
        if (checkClientCanUse(clients)) { // double check，再次检测客户端是否全部可用
            batchClientRefIncr(clients); // 增加应用Client的次数
            return clients;
        }
        connectNum = Math.max(connectNum, 1); // 至少一个共享连接
        // 如果当前Clients集合为空，则直接通过initClient()方法初始化所有共享客户端
        if (CollectionUtils.isEmpty(clients)) {
            clients = buildReferenceCountExchangeClientList(url, connectNum);
            referenceClientMap.put(key, clients);
        } else { // 如果只有部分共享客户端不可用，则只需要处理这些不可用的客户端
            for (int i = 0; i < clients.size(); i++) {
                ReferenceCountExchangeClient referenceCountExchangeClient = clients.get(i);
                if (referenceCountExchangeClient == null || referenceCountExchangeClient.isClosed()) {
                    clients.set(i, buildReferenceCountExchangeClient(url));
                    continue;
                }
                // 增加引用
                referenceCountExchangeClient.incrementAndGetCount();
            }
        }
        // 清理locks集合中的锁对象，防止内存泄漏，如果key对应的服务宕机或是下线，
        // 这里不进行清理的话，这个用于加锁的Object对象是无法被GC的，从而出现内存泄漏
        locks.remove(key); 
        return clients;
    }
}
```

这里使用的 ExchangeClient 实现是 ReferenceCountExchangeClient,它是 ExchangeClient 的一个装饰器,在原始 ExchangeClient 对象基础上添加了引用计数的功能.<br>

ReferenceCountExchangeClient 中除了持有被修饰的 ExchangeClient 对象外,还有一个 referenceCount 字段(AtomicInteger 类型),用于记录该 Client 被应用的次数.从下图中我们可以看到,在 ReferenceCountExchangeClient 的构造方法以及 incrementAndGetCount() 方法中会增加引用次数,在 close() 方法中则会减少引用次数.<br>

这样,于同一个地址的共享连接,就可以满足两个基本需求:
* 当引用次数减到 0 的时候,ExchangeClient 连接关闭
* 当引用次数未减到 0 的时候,底层的 ExchangeClient 不能关闭

还有一个需要注意的细节是 ReferenceCountExchangeClient.close() 方法,在关闭底层 ExchangeClient 对象之后,会立即创建一个 LazyConnectExchangeClient ,也有人称其为"幽灵连接".具体逻辑如下所示,这里的 LazyConnectExchangeClient 主要用于异常情况的兜底:
```
public void close(int timeout) {

    // 引用次数减到0，关闭底层的ExchangeClient，具体操作有：停掉心跳任务、重连任务以及关闭底层Channel，这些在前文介绍HeaderExchangeClient的时候已经详细分析过了，这里不再赘述
    if (referenceCount.decrementAndGet() <= 0) { 
        if (timeout == 0) {
            client.close();
        } else {
            client.close(timeout);
        } 
        // 创建LazyConnectExchangeClient，并将client字段指向该对象
        replaceWithLazyClient(); 
    }
}

private void replaceWithLazyClient() {
    // 在原有的URL之上，添加一些LazyConnectExchangeClient特有的参数
    URL lazyUrl = URLBuilder.from(url)
            .addParameter(LAZY_CONNECT_INITIAL_STATE_KEY, Boolean.TRUE)
            .addParameter(RECONNECT_KEY, Boolean.FALSE)
            .addParameter(SEND_RECONNECT_KEY, Boolean.TRUE.toString())
            .addParameter("warning", Boolean.TRUE.toString())
            .addParameter(LazyConnectExchangeClient.REQUEST_WITH_WARNING_KEY, true)
            .addParameter("_client_memo", "referencecounthandler.replacewithlazyclient")
            .build();
    // 如果当前client字段已经指向了LazyConnectExchangeClient，则不需要再次创建LazyConnectExchangeClient兜底了
    if (!(client instanceof LazyConnectExchangeClient) || client.isClosed()) {
        // ChannelHandler依旧使用原始ExchangeClient使用的Handler，即DubboProtocol中的requestHandler字段
        client = new LazyConnectExchangeClient(lazyUrl, client.getExchangeHandler());
    }
}
```

LazyConnectExchangeClient 也是 ExchangeClient 的装饰器,它会在原有 ExchangeClient 对象的基础上添加懒加载的功能.LazyConnectExchangeClient 在构造方法中不会创建底层持有连接的 Client,而是在需要发送请求的时候,才会调用 initClient() 方法进行 Client 的创建<br>


<br>
**<font size="3">创建独享连接</font>** <br>

创建独享 Client 的入口在DubboProtocol.initClient() 方法,它首先会在 URL 中设置一些默认的参数,然后根据 LAZY_CONNECT_KEY 参数决定是否使用 LazyConnectExchangeClient 进行封装,实现懒加载功能,如下代码所示
```
private ExchangeClient initClient(URL url) {

    // 获取客户端扩展名并进行检查，省略检测的逻辑
    String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
    // 设置Codec2的扩展名
    url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
    // 设置默认的心跳间隔
    url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));
    ExchangeClient client;    
    // 如果配置了延迟创建连接的特性，则创建LazyConnectExchangeClient
    if (url.getParameter(LAZY_CONNECT_KEY, false)) {
        client = new LazyConnectExchangeClient(url, requestHandler);
    } else { // 未使用延迟连接功能，则直接创建HeaderExchangeClient
        client = Exchangers.connect(url, requestHandler);
    }
    return client;
}
```


<br><br>
## <span id="jump4">四. destroy方法</span>

在 DubboProtocol 销毁的时候,会调用 destroy() 方法释放底层资源,其中就涉及 export 流程中创建的 ProtocolServer 对象以及 refer 流程中创建的 Client.<br>

DubboProtocol.destroy() 方法首先会逐个关闭 serverMap 集合中的 ProtocolServer 对象,相关代码片段如下
```
for (String key : new ArrayList<>(serverMap.keySet())) {

    ProtocolServer protocolServer = serverMap.remove(key);
    if (protocolServer == null) { continue;}
    RemotingServer server = protocolServer.getRemotingServer();
    // 在close()方法中，发送ReadOnly请求、阻塞指定时间、关闭底层的定时任务、关闭相关线程池，最终，会断开所有连接，关闭Server。这些逻辑在前文介绍HeaderExchangeServer、NettyServer等实现的时候，已经详细分析过了，这里不再展开
    server.close(ConfigurationUtils.getServerShutdownTimeout());
}
```

ConfigurationUtils.getServerShutdownTimeout() 方法返回的阻塞时长默认是 10 秒,可以通过 dubbo.service.shutdown.wait 或是 dubbo.service.shutdown.wait.seconds 进行配置.<br>

之后,DubboProtocol.destroy() 方法会逐个关闭 referenceClientMap 集合中的 Client,逻辑与上述关闭ProtocolServer的逻辑相同,这里不再重复.只不过需要注意前面我们提到的 ReferenceCountExchangeClient 的存在,只有引用减到 0,底层的 Client 才会真正销毁.<br>

最后,DubboProtocol.destroy() 方法会调用父类 AbstractProtocol 的 destroy() 方法,销毁全部 Invoker 对象.