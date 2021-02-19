---
layout:     post
title:      "Dubbo 集群容错"
date:       2021-02-19 13:15:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---



## 导航
[一. 集群容错的背景和意义](#jump1)
<br>
[二. Cluster 接口与容错机制](#jump2)
<br>
[三. AbstractClusterInvoker](#jump3)
<br>
[四. AbstractCluster](#jump4)
<br>
[五. FailoverClusterInvoker](#jump5)
<br>
[六. FailbackClusterInvoker](#jump6)
<br>
[七. FailfastClusterInvoker](#jump7)
<br>
[八. FailsafeClusterInvoker](#jump8)
<br>
[九. ForkingClusterInvoker](#jump9)
<br>
[十. BroadcastClusterInvoker](#jump10)
<br>
[十一. AvailableClusterInvoker](#jump11)
<br>
[十二. MergeableClusterInvoker](#jump12)
<br>
[十三. ZoneAwareClusterInvoker](#jump13)
<br>







<br><br>
## <span id="jump1">一. 集群容错的背景和意义</span>

Cluster 接口提供了我们常说的集群容错功能.<br>

集群中的单个节点有一定概率出现一些问题,例如,磁盘损坏、系统崩溃等,导致节点无法对外提供服务,因此在分布式 RPC 框架中,必须要重视这种情况.为了避免单点故障,我们的 Provider 通常至少会部署在两台服务器上,以集群的形式对外提供服务,对于一些负载比较高的服务,则需要部署更多 Provider 来抗住流量.<br>

在 Dubbo 中,通过 Cluster 这个接口把一组可供调用的 Provider 信息组合成为一个统一的 Invoker 供调用方进行调用.经过 Router 过滤、LoadBalance 选址之后,选中一个具体 Provider 进行调用,如果调用失败,则会按照集群的容错策略进行容错处理.<br>

Dubbo 默认内置了若干容错策略,并且每种容错策略都有自己独特的应用场景,我们可以通过配置选择不同的容错策略.如果这些内置容错策略不能满足需求,我们还可以通过自定义容错策略进行配置.<br>



<br><br>
## <span id="jump2">二. Cluster 接口与容错机制</span>

Cluster 的工作流程大致可以分为两步(如下图所示):
1. 创建 Cluster Invoker 实例(在 Consumer 初始化时,Cluster 实现类会创建一个 Cluster Invoker 实例,即下图中的 merge 操作)
2. 使用 Cluster Invoker 实例(在 Consumer 服务消费者发起远程调用请求的时候,Cluster Invoker 会依赖前面课时介绍的 Directory、Router、LoadBalance 等组件得到最终要调用的 Invoker 对象)

[![yf5B79.png](https://s3.ax1x.com/2021/02/19/yf5B79.png)](https://imgchr.com/i/yf5B79)

Cluster Invoker 获取 Invoker 的流程大致可描述为如下:
1. 通过 Directory 获取 Invoker 列表,以 RegistryDirectory 为例,会感知注册中心的动态变化,实时获取当前 Provider 对应的 Invoker 集合
2. 调用 Router 的 route() 方法进行路由,过滤掉不符合路由规则的 Invoker 对象
3. 通过 LoadBalance 从 Invoker 列表中选择一个 Invoker
4. ClusterInvoker 会将参数传给 LoadBalance 选择出的 Invoker 实例的 invoke 方法,进行真正的远程调用

这个过程是一个正常流程,没有涉及容错处理.Dubbo 中常见的容错方式有如下几个
* Failover Cluster: 失败自动切换.它是 Dubbo 的默认容错机制,在请求一个 Provider 节点失败的时候,自动切换其他 Provider 节点,默认执行 3 次,适合幂等操作.当然,重试次数越多,在故障容错的时候带给 Provider 的压力就越大,在极端情况下甚至可能造成雪崩式的问题
* Failback Cluster: 失败自动恢复.失败后记录到队列中, 通过定时器重试
* Failfast Cluster: 快速失败.请求失败后返回异常, 不进行任何重试
* Failsafe Cluster: 失败安全.请求失败后忽略异常, 不进行任何重试
* Forking Cluster: 并行调用多个 Provider 节点, 只要有一个成功就返回
* Broadcast Cluster: 广播多个 Provider 节点, 只要有一个节点失败就失败
* Available Cluster: 遍历所有的 Provider 节点,找到每一个可用的节点,就直接调用.如果没有可用的 Provider 节点,则直接抛出异常
* Mergeable Cluster: 请求多个 Provider 节点并将得到的结果进行合并

下面我们再来看 Cluster 接口.Cluster 接口是一个扩展接口,通过 @SPI 注解的参数我们知道其使用的默认实现是 FailoverCluster,它只定义了一个 join() 方法,在其上添加了 @Adaptive 注解,会动态生成适配器类,其中会优先根据 Directory.getUrl() 方法返回的 URL 中的 cluster 参数值选择扩展实现,若无 cluster 参数则使用默认的 FailoverCluster 实现.Cluster 接口的具体定义如下所示
```
@SPI(FailoverCluster.NAME)
public interface Cluster {
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}
```

Cluster 接口的实现类如下图所示,分别对应前面提到的多种容错策略:
[![yhCp1U.png](https://s3.ax1x.com/2021/02/19/yhCp1U.png)](https://imgchr.com/i/yhCp1U)

在每个 Cluster 接口实现中,都会创建对应的 Invoker 对象,这些都继承自 AbstractClusterInvoker 抽象类,如下图所示
[![yhCt9f.png](https://s3.ax1x.com/2021/02/19/yhCt9f.png)](https://imgchr.com/i/yhCt9f)

通过上面两张继承关系图我们可以看出,Cluster 接口和 Invoker 接口都会有相应的抽象实现类,这些抽象实现类都实现了一些公共能力.下面我们就来深入介绍 AbstractClusterInvoker 和 AbstractCluster 这两个抽象类



<br><br>
## <span id="jump3">三. AbstractClusterInvoker</span>

AbstractClusterInvoker,它有两点核心功能:一个是实现的 Invoker 接口,对 Invoker.invoke() 方法进行通用的抽象实现;另一个是实现通用的负载均衡算法.<br>

在 AbstractClusterInvoker.invoke() 方法中,会通过 Directory 获取 Invoker 列表,然后通过 SPI 初始化 LoadBalance,最后调用 doInvoke() 方法执行子类的逻辑.在 Directory.list() 方法返回 Invoker 集合之前,已经使用 Router 进行了一次筛选,可以回顾之前对 RegistryDirectory 的分析
```
public Result invoke(final Invocation invocation) throws RpcException {
    // 检测当前Invoker是否已销毁
    checkWhetherDestroyed(); 
    // 将RpcContext中的attachment添加到Invocation中
    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
    }
    // 通过Directory获取Invoker对象列表，通过对RegistryDirectory的介绍我们知道，其中已经调用了Router进行过滤
    List<Invoker<T>> invokers = list(invocation);
    // 通过SPI加载LoadBalance
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    // 调用doInvoke()方法，该方法是个抽象方法
    return doInvoke(invocation, invokers, loadbalance);
}

protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
    return directory.list(invocation); // 调用Directory.list()方法
}
```

下面我们来看一下 AbstractClusterInvoker 是如何按照不同的 LoadBalance 算法从 Invoker 集合中选取最终 Invoker 对象的.<br>

AbstractClusterInvoker 并没有简单粗暴地使用 LoadBalance.select() 方法完成负载均衡,而是做了进一步的封装,具体实现在 select() 方法中.在 select() 方法中会根据配置决定是否开启粘滞连接特性,如果开启了,则需要将上次使用的 Invoker 缓存起来,只要 Provider 节点可用就直接调用,不会再进行负载均衡.如果调用失败,才会重新进行负载均衡,并且排除已经重试过的 Provider 节点.粘滞连接特性默认关闭.
```
// 第一个参数是此次使用的LoadBalance实现，第二个参数Invocation是此次服务调用的上下文信息，
// 第三个参数是待选择的Invoker集合，第四个参数用来记录负载均衡已经选出来、尝试过的Invoker集合
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    // 获取调用方法名
    String methodName = invocation == null ? StringUtils.EMPTY_STRING : invocation.getMethodName();
    // 获取sticky配置，sticky表示粘滞连接，所谓粘滞连接是指Consumer会尽可能地
    // 调用同一个Provider节点，除非这个Provider无法提供服务
    boolean sticky = invokers.get(0).getUrl()
            .getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);
    // 检测invokers列表是否包含sticky Invoker，如果不包含，
    // 说明stickyInvoker代表的服务提供者挂了，此时需要将其置空
    if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
        stickyInvoker = null;
    }
    // 如果开启了粘滞连接特性，需要先判断这个Provider节点是否已经重试过了
    if (sticky && stickyInvoker != null // 表示粘滞连接
            && (selected == null || !selected.contains(stickyInvoker)) // 表示stickyInvoker未重试过
    ) {
        // 检测当前stickyInvoker是否可用，如果可用，直接返回stickyInvoker
        if (availablecheck && stickyInvoker.isAvailable()) { 
            return stickyInvoker;
        }
    }
    // 执行到这里，说明前面的stickyInvoker为空，或者不可用
    // 这里会继续调用doSelect选择新的Invoker对象
    Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);
    if (sticky) { // 是否开启粘滞，更新stickyInvoker字段
        stickyInvoker = invoker;
    }
    return invoker;
}
```

doSelect() 方法主要做了两件事:
* 一是通过 LoadBalance 选择 Invoker 对象
* 二是如果选出来的 Invoker 不稳定或不可用,会调用 reselect() 方法进行重选

```
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    // 判断是否需要进行负载均衡，Invoker集合为空，直接返回null
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    if (invokers.size() == 1) { // 只有一个Invoker对象，直接返回即可
        return invokers.get(0);
    }
    // 通过LoadBalance实现选择Invoker对象
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
    // 如果LoadBalance选出的Invoker对象，已经尝试过请求了或不可用，则需要调用reselect()方法重选
    if ((selected != null && selected.contains(invoker)) // Invoker已经尝试调用过了，但是失败了
            || (!invoker.isAvailable() && getUrl() != null && availablecheck) // Invoker不可用
    ) {
        try {
            // 调用reselect()方法重选
            Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            // 如果重选的Invoker对象不为空，则直接返回这个 rInvoker
            if (rInvoker != null) {
                invoker = rInvoker;
            } else {
                int index = invokers.indexOf(invoker);
                try {
                    // 如果重选的Invoker对象为空，则返回该Invoker的下一个Invoker对象
                    invoker = invokers.get((index + 1) % invokers.size());
                } catch (Exception e) {
                    logger.warn("...");
                }
            }
        } catch (Throwable t) {
            logger.error("...");
        }
    }
    return invoker;
}
```

reselect() 方法会重新进行一次负载均衡,首先对未尝试过的可用 Invokers 进行负载均衡,如果已经全部重试过了,则将尝试过的 Provider 节点过滤掉,然后在可用的 Provider 节点中重新进行负载均衡
```
private Invoker<T> reselect(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected, boolean availablecheck) throws RpcException {
    // 用于记录要重新进行负载均衡的Invoker集合
    List<Invoker<T>> reselectInvokers = new ArrayList<>(
            invokers.size() > 1 ? (invokers.size() - 1) : invokers.size());
    // 将不在selected集合中的Invoker过滤出来进行负载均衡
    for (Invoker<T> invoker : invokers) {
        if (availablecheck && !invoker.isAvailable()) {
            continue;
        }
        if (selected == null || !selected.contains(invoker)) {
            reselectInvokers.add(invoker);
        }
    }
    // reselectInvokers不为空时，才需要通过负载均衡组件进行选择
    if (!reselectInvokers.isEmpty()) {
        return loadbalance.select(reselectInvokers, getUrl(), invocation);
    }
    // 只能对selected集合中可用的Invoker再次进行负载均衡
    if (selected != null) {
        for (Invoker<T> invoker : selected) {
            if ((invoker.isAvailable()) // available first
                    && !reselectInvokers.contains(invoker)) {
                reselectInvokers.add(invoker);
            }
        }
    }
    if (!reselectInvokers.isEmpty()) {
        return loadbalance.select(reselectInvokers, getUrl(), invocation);
    }
    return null;
}
```



<br><br>
## <span id="jump4">四. AbstractCluster</span>

Cluster 扩展实现都继承了 AbstractCluster 抽象类.AbstractCluster 抽象类的核心逻辑是在 ClusterInvoker 外层包装一层 ClusterInterceptor,从而实现类似切面的效果.<br>

下面是 ClusterInterceptor 接口的定义
```
@SPI
public interface ClusterInterceptor {
    // 前置拦截方法
    void before(AbstractClusterInvoker<?> clusterInvoker, Invocation invocation);
    // 后置拦截方法
    void after(AbstractClusterInvoker<?> clusterInvoker, Invocation invocation);
    // 调用ClusterInvoker的invoke()方法完成请求
    default Result intercept(AbstractClusterInvoker<?> clusterInvoker, Invocation invocation) throws RpcException {
        return clusterInvoker.invoke(invocation);
    }
    // 这个Listener用来监听请求的正常结果以及异常
    interface Listener {
        void onMessage(Result appResponse, AbstractClusterInvoker<?> clusterInvoker, Invocation invocation);
        void onError(Throwable t, AbstractClusterInvoker<?> clusterInvoker, Invocation invocation);
    }
}
```

在 AbstractCluster 抽象类的 join() 方法中,首先会调用 doJoin() 方法获取最终要调用的 Invoker 对象,doJoin() 是个抽象方法,由 AbstractCluster 子类根据具体的策略进行实现.之后,AbstractCluster.join() 方法会调用 buildClusterInterceptors() 方法加载 ClusterInterceptor 扩展实现类,对 Invoker 对象进行包装.具体实现如下
```
private <T> Invoker<T> buildClusterInterceptors(AbstractClusterInvoker<T> clusterInvoker, String key) {
    AbstractClusterInvoker<T> last = clusterInvoker;
    // 通过SPI方式加载ClusterInterceptor扩展实现
    List<ClusterInterceptor> interceptors = ExtensionLoader.getExtensionLoader(ClusterInterceptor.class).getActivateExtension(clusterInvoker.getUrl(), key);
    if (!interceptors.isEmpty()) {
        for (int i = interceptors.size() - 1; i >= 0; i--) {
            // 将InterceptorInvokerNode收尾连接到一起，形成调用链
            final ClusterInterceptor interceptor = interceptors.get(i);
            final AbstractClusterInvoker<T> next = last;
            last = new InterceptorInvokerNode<>(clusterInvoker, interceptor, next);
        }
    }
    return last;
}

@Override
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    // 扩展名称由reference.interceptor参数确定
    return buildClusterInterceptors(doJoin(directory), directory.getUrl().getParameter(REFERENCE_INTERCEPTOR_KEY));
}
```

InterceptorInvokerNode 会将底层的 AbstractClusterInvoker 对象以及关联的 ClusterInterceptor 对象封装到一起,还会维护一个 next 引用,指向下一个 InterceptorInvokerNode 对象.<br>

在 InterceptorInvokerNode.invoke() 方法中,会先调用 ClusterInterceptor 的前置逻辑,然后执行 intercept() 方法调用 AbstractClusterInvoker 的 invoke() 方法完成远程调用,最后执行 ClusterInterceptor 的后置逻辑.具体实现如下:
```
public Result invoke(Invocation invocation) throws RpcException {
    Result asyncResult;
    try {
        interceptor.before(next, invocation); // 前置逻辑
        // 执行invoke()方法完成远程调用
        asyncResult = interceptor.intercept(next, invocation);
    } catch (Exception e) {
        if (interceptor instanceof ClusterInterceptor.Listener) {
            // 出现异常时，会触发监听器的onError()方法
            ClusterInterceptor.Listener listener = (ClusterInterceptor.Listener) interceptor;
            listener.onError(e, clusterInvoker, invocation);
        }
        throw e;
    } finally {
        // 执行后置逻辑
        interceptor.after(next, invocation);
    }
    return asyncResult.whenCompleteWithContext((r, t) -> {
        if (interceptor instanceof ClusterInterceptor.Listener) {
            ClusterInterceptor.Listener listener = (ClusterInterceptor.Listener) interceptor;
            if (t == null) {
                // 正常返回时，会调用onMessage()方法触发监听器
                listener.onMessage(r, clusterInvoker, invocation);
            } else {
                listener.onError(t, clusterInvoker, invocation);
            }
        }
    });
}
```

Dubbo 提供了两个 ClusterInterceptor 实现类,分别是 ConsumerContextClusterInterceptor 和 ZoneAwareClusterInterceptor,如下图所示
[![yhe8ED.png](https://s3.ax1x.com/2021/02/19/yhe8ED.png)](https://imgchr.com/i/yhe8ED)

在 ConsumerContextClusterInterceptor 的 before() 方法中,会在 RpcContext 中设置当前 Consumer 地址、此次调用的 Invoker 等信息,同时还会删除之前与当前线程绑定的 Server Context.在 after() 方法中,会删除本地 RpcContext 的信息.ConsumerContextClusterInterceptor 的具体实现如下
```
public void before(AbstractClusterInvoker<?> invoker, Invocation invocation) {
    // 获取当前线程绑定的RpcContext
    RpcContext context = RpcContext.getContext();
    // 设置Invoker、Consumer地址等信息 context.setInvocation(invocation).setLocalAddress(NetUtils.getLocalHost(), 0);
    if (invocation instanceof RpcInvocation) {
        ((RpcInvocation) invocation).setInvoker(invoker);
    }
    RpcContext.removeServerContext();
}

public void after(AbstractClusterInvoker<?> clusterInvoker, Invocation invocation) {
    RpcContext.removeContext(true); // 删除本地RpcContext的信息
}
```

ConsumerContextClusterInterceptor 同时继承了 ClusterInterceptor.Listener 接口,在其 onMessage() 方法中,会获取响应中的 attachments 并设置到 RpcContext 中的 SERVER_LOCAL 之中,具体实现如下
```
public void onMessage(Result appResponse, AbstractClusterInvoker<?> invoker, Invocation invocation) {
// 从AppResponse中获取attachment，并设置到SERVER_LOCAL这个RpcContext中    
RpcContext.getServerContext().setObjectAttachments(appResponse.getObjectAttachments());
}
```

介绍完 ConsumerContextClusterInterceptor,我们再来看 ZoneAwareClusterInterceptor.<br>

在 ZoneAwareClusterInterceptor 的 before() 方法中,会从 RpcContext 中获取多注册中心相关的参数并设置到 Invocation 中(主要是 registry_zone 参数和 registry_zone_force 参数,这两个参数的具体含义，在后面分析 ZoneAwareClusterInvoker 时详细介绍),ZoneAwareClusterInterceptor 的 after() 方法为空实现.ZoneAwareClusterInterceptor 的具体实现如下
```
public void before(AbstractClusterInvoker<?> clusterInvoker, Invocation invocation) {
    RpcContext rpcContext = RpcContext.getContext();
    // 从RpcContext中获取registry_zone参数和registry_zone_force参数
    String zone = (String) rpcContext.getAttachment(REGISTRY_ZONE);
    String force = (String) rpcContext.getAttachment(REGISTRY_ZONE_FORCE);
    // 检测用户是否提供了ZoneDetector接口的扩展实现
    ExtensionLoader<ZoneDetector> loader = ExtensionLoader.getExtensionLoader(ZoneDetector.class);
    if (StringUtils.isEmpty(zone) && loader.hasExtension("default")) {
        ZoneDetector detector = loader.getExtension("default");
        zone = detector.getZoneOfCurrentRequest(invocation);
        force = detector.isZoneForcingEnabled(invocation, zone);
    }
    // 将registry_zone参数和registry_zone_force参数设置到Invocation中
    if (StringUtils.isNotEmpty(zone)) {
        invocation.setAttachment(REGISTRY_ZONE, zone);
    }
    if (StringUtils.isNotEmpty(force)) {
        invocation.setAttachment(REGISTRY_ZONE_FORCE, force);
    }
}
```
ZoneAwareClusterInterceptor 没有实现 ClusterInterceptor.Listener 接口,也就是不提供监听响应的功能.<br>



<br><br>
## <span id="jump5">五. FailoverClusterInvoker</span>

Cluster 接口默认的扩展实现是 FailoverCluster,其 doJoin() 方法中会创建一个 FailoverClusterInvoker 对象并返回,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new FailoverClusterInvoker<>(directory);
}
```

FailoverClusterInvoker 会在调用失败的时候,自动切换 Invoker 进行重试.下面来看 FailoverClusterInvoker 的核心实现
```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyInvokers = invokers;
    // 检查copyInvokers集合是否为空，如果为空会抛出异常
    checkInvokers(copyInvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    // 参数重试次数，默认重试2次，总共执行3次
    int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    RpcException le = null;
    // 记录已经尝试调用过的Invoker对象
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size());
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        // 第一次传进来的invokers已经check过了，第二次则是重试，需要重新获取最新的服务列表
        if (i > 0) {
            checkWhetherDestroyed();
            // 这里会重新调用Directory.list()方法，获取Invoker列表
            copyInvokers = list(invocation);
            // 检查copyInvokers集合是否为空，如果为空会抛出异常
            checkInvokers(copyInvokers, invocation);
        }
        // 通过LoadBalance选择Invoker对象，这里传入的invoked集合，
        // 就是前面介绍AbstractClusterInvoker.select()方法中的selected集合
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        // 记录此次要尝试调用的Invoker对象，下一次重试时就会过滤这个服务
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            // 调用目标Invoker对象的invoke()方法，完成远程调用
            Result result = invoker.invoke(invocation);
            // 经过尝试之后，终于成功，这里会打印一个警告日志，将尝试过来的Provider地址打印出来
            if (le != null && logger.isWarnEnabled()) {
                logger.warn("...");
            }
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) { // 抛出异常，表示此次尝试失败，会进行重试
            le = new RpcException(e.getMessage(), e);
        } finally {
            // 记录尝试过的Provider地址，会在上面的警告日志中打印出来
            providers.add(invoker.getUrl().getAddress());
        }
    }
    // 达到重试次数上限之后，会抛出异常，其中会携带调用的方法名、尝试过的Provider节点的地址(providers集合)、全部的Provider个数(copyInvokers集合)以及Directory信息
    throw new RpcException(le.getCode(), "...");
}
```



<br><br>
## <span id="jump5">六. FailbackClusterInvoker</span>

FailbackCluster 是 Cluster 接口的另一个扩展实现,扩展名是 failback,其 doJoin() 方法中创建的 Invoker 对象是 FailbackClusterInvoker 类型,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new FailbackClusterInvoker<>(directory);
}
```

FailbackClusterInvoker 在请求失败之后,返回一个空结果给 Consumer,同时还会添加一个定时任务对失败的请求进行重试.下面来看 FailbackClusterInvoker 的具体实现
```
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    Invoker<T> invoker = null;
    try {
        // 检测Invoker集合是否为空
        checkInvokers(invokers, invocation);
        // 调用select()方法得到此次尝试的Invoker对象
        invoker = select(loadbalance, invocation, invokers, null);
        // 调用invoke()方法完成远程调用
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        // 请求失败之后，会添加一个定时任务进行重试
        addFailed(loadbalance, invocation, invokers, invoker);
        return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // 请求失败时，会返回一个空结果
    }
}
```

在 doInvoke() 方法中,请求失败时会调用 addFailed() 方法添加定时任务进行重试,默认每隔 5 秒执行一次,总共重试 3 次,具体实现如下
```
private void addFailed(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker) {
    if (failTimer == null) {
        synchronized (this) {
            if (failTimer == null) { // Double Check防止并发问题 
                // 初始化时间轮，这个时间轮有32个槽，每个槽代表1秒
                failTimer = new HashedWheelTimer(
                        new NamedThreadFactory("failback-cluster-timer", true),
                        1,
                        TimeUnit.SECONDS, 32, failbackTasks);
            }
        }
    }
    // 创建一个定时任务
    RetryTimerTask retryTimerTask = new RetryTimerTask(loadbalance, invocation, invokers, lastInvoker, retries, RETRY_FAILED_PERIOD);
    try {
        // 将定时任务添加到时间轮中
        failTimer.newTimeout(retryTimerTask, RETRY_FAILED_PERIOD, TimeUnit.SECONDS);
    } catch (Throwable e) {
        logger.error("...");
    }
}
```

在 RetryTimerTask 定时任务中,会重新调用 select() 方法筛选合适的 Invoker 对象,并尝试进行请求.如果请求再次失败且重试次数未达到上限,则调用 rePut() 方法再次添加定时任务,等待进行重试;如果请求成功,也不会返回任何结果.RetryTimerTask 的核心实现如下
```
public void run(Timeout timeout) {
    try {
        // 重新选择Invoker对象，注意，这里会将上次重试失败的Invoker作为selected集合传入
        Invoker<T> retryInvoker = select(loadbalance, invocation, invokers, Collections.singletonList(lastInvoker));
        lastInvoker = retryInvoker;
        retryInvoker.invoke(invocation); // 请求对应的Provider节点
    } catch (Throwable e) {
        if ((++retryTimes) >= retries) { // 重试次数达到上限，输出警告日志
            logger.error("...");
        } else {
            rePut(timeout); // 重试次数未达到上限，则重新添加定时任务，等待重试
        }
    }
}

private void rePut(Timeout timeout) {
    if (timeout == null) { // 边界检查
        return;
    }
    Timer timer = timeout.timer();
    if (timer.isStop() || timeout.isCancelled()) { // 检查时间轮状态、检查定时任务状态
        return;
    }
    // 重新添加定时任务
    timer.newTimeout(timeout.task(), tick, TimeUnit.SECONDS);
}
```



<br><br>
## <span id="jump7">七. FailfastClusterInvoker</span>

FailfastCluster 的扩展名是 failfast,在其 doJoin() 方法中会创建 FailfastClusterInvoker 对象,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new FailfastClusterInvoker<>(directory);
}
```

FailfastClusterInvoker 只会进行一次请求,请求失败之后会立即抛出异常,这种策略适合非幂等的操作,具体实现如下
```
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    // 调用select()得到此次要调用的Invoker对象
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        return invoker.invoke(invocation); // 发起请求
    } catch (Throwable e) { 
        // 请求失败，直接抛出异常
        if (e instanceof RpcException && ((RpcException) e).isBiz()) {
            throw (RpcException) e;
        }
        throw new RpcException("...");
    }
}
```



<br><br>
## <span id="jump8">八. FailsafeClusterInvoker</span>

FailsafeCluster 的扩展名是 failsafe,在其 doJoin() 方法中会创建 FailsafeClusterInvoker 对象,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new FailsafeClusterInvoker<>(directory);
}
```

FailsafeClusterInvoker 只会进行一次请求,请求失败之后会返回一个空结果,具体实现如下
```
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        // 检测Invoker集合是否为空
        checkInvokers(invokers, invocation);
        // 调用select()得到此次要调用的Invoker对象
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        // 发起请求
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        // 请求失败之后，会打印一行日志并返回空结果
        logger.error("...");
        return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); 
    }
}
```



<br><br>
## <span id="jump9">九. ForkingClusterInvoker</span>

ForkingCluster 的扩展名称为 forking,在其 doJoin() 方法中,会创建一个 ForkingClusterInvoker 对象,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new ForkingClusterInvoker<>(directory);
}
```

ForkingClusterInvoker 中会维护一个线程池(executor 字段,通过 Executors.newCachedThreadPool() 方法创建的线程池),并发调用多个 Provider 节点,只要有一个 Provider 节点成功返回了结果,ForkingClusterInvoker 的 doInvoke()方法就会立即结束运行.<br>

ForkingClusterInvoker 主要是为了应对一些实时性要求较高的读操作,因为没有并发控制的多线程写入,可能会导致数据不一致.<br>

ForkingClusterInvoker.doInvoke() 方法首先从 Invoker 集合中选出指定个数(forks 参数决定)的 Invoker 对象,然后通过 executor 线程池并发调用这些 Invoker,并将请求结果存储在 ref 阻塞队列中,则当前线程会阻塞在 ref 队列上,等待第一个请求结果返回.下面是 ForkingClusterInvoker 的具体实现
```
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        // 检查Invoker集合是否为空
        checkInvokers(invokers, invocation);
        final List<Invoker<T>> selected;
        // 从URL中获取forks参数，作为并发请求的上限，默认值为2
        final int forks = getUrl().getParameter(FORKS_KEY, DEFAULT_FORKS);
        final int timeout = getUrl().getParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
        if (forks <= 0 || forks >= invokers.size()) {
            // 如果forks为负数或是大于Invoker集合的长度，会直接并发调用全部Invoker
            selected = invokers;
        } else {
            // 按照forks指定的并发度，选择此次并发调用的Invoker对象
            selected = new ArrayList<>(forks);
            while (selected.size() < forks) {
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if (!selected.contains(invoker)) {
                    selected.add(invoker); // 避免重复选择
                }
            }
        }
        RpcContext.getContext().setInvokers((List) selected);
        // 记录失败的请求个数
        final AtomicInteger count = new AtomicInteger();
        // 用于记录请求结果
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
        for (final Invoker<T> invoker : selected) {  // 遍历 selected 列表
            executor.execute(() -> { // 为每个Invoker创建一个任务，并提交到线程池中
                try {
                    // 发起请求
                    Result result = invoker.invoke(invocation);
                    // 将请求结果写到ref队列中
                    ref.offer(result);
                } catch (Throwable e) {
                    int value = count.incrementAndGet();
                    if (value >= selected.size()) {
                        // 如果失败的请求个数超过了并发请求的个数，则向ref队列中写入异常
                        ref.offer(e); 
                    }
                }
            });
        }
        try {
            // 当前线程会阻塞等待任意一个请求结果的出现
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            if (ret instanceof Throwable) { // 如果结果类型为Throwable，则抛出异常
                Throwable e = (Throwable) ret;
                throw new RpcException("...");
            }
            return (Result) ret; // 返回结果
        } catch (InterruptedException e) {
             throw new RpcException("...");
        }
    } finally { 
        // 清除上下文信息
        RpcContext.getContext().clearAttachments();
    }
}
```



<br><br>
## <span id="jump10">十. BroadcastClusterInvoker</span>

BroadcastCluster 这个 Cluster 实现类的扩展名为 broadcast,在其 doJoin() 方法中创建的是 BroadcastClusterInvoker 类型的 Invoker 对象,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new BroadcastClusterInvoker<>(directory);
}
```

在 BroadcastClusterInvoker 中,会逐个调用每个 Provider 节点,其中任意一个 Provider 节点报错,都会在全部调用结束之后抛出异常.BroadcastClusterInvoker通常用于通知类的操作,例如通知所有 Provider 节点更新本地缓存<br>

下面来看 BroadcastClusterInvoker 的具体实现
```
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    // 检测Invoker集合是否为空
    checkInvokers(invokers, invocation);
    RpcContext.getContext().setInvokers((List) invokers);
    RpcException exception = null; // 用于记录失败请求的相关异常信息
    Result result = null;
    // 遍历所有Invoker对象
    for (Invoker<T> invoker : invokers) {
        try {
            // 发起请求
            result = invoker.invoke(invocation);
        } catch (RpcException e) {
            exception = e;
            logger.warn(e.getMessage(), e);
        } catch (Throwable e) {
            exception = new RpcException(e.getMessage(), e);
            logger.warn(e.getMessage(), e);
        }
    }
    if (exception != null) { // 出现任何异常，都会在这里抛出
        throw exception;
    }
    return result;
}
```



<br><br>
## <span id="jump11">十一. AvailableClusterInvoker</span>

AvailableCluster 这个 Cluster 实现类的扩展名为 available,在其 join() 方法中创建的是 AvailableClusterInvoker 类型的 Invoker 对象,具体实现如下
```
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new AvailableClusterInvoker<>(directory);
}
```

在 AvailableClusterInvoker 的 doInvoke() 方法中,会遍历整个 Invoker 集合,逐个调用对应的 Provider 节点,当遇到第一个可用的 Provider 节点时,就尝试访问该 Provider 节点,成功则返回结果;如果访问失败,则抛出异常终止遍历<br>

下面是 AvailableClusterInvoker 的具体实现
```
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    for (Invoker<T> invoker : invokers) { // 遍历整个Invoker集合
        if (invoker.isAvailable()) { // 检测该Invoker是否可用
            // 发起请求，调用失败时的异常会直接抛出
            return invoker.invoke(invocation);
        }
    }
    // 没有找到可用的Invoker，也会抛出异常
    throw new RpcException("No provider available in " + invokers);
}
```



<br><br>
## <span id="jump12">十二. MergeableClusterInvoker</span>

MergeableCluster 这个 Cluster 实现类的扩展名为 mergeable,在其 doJoin() 方法中创建的是 MergeableClusterInvoker 类型的 Invoker 对象,具体实现如下
```
public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new MergeableClusterInvoker<T>(directory);
}
```

MergeableClusterInvoker 会对多个 Provider 节点返回结果合并.如果请求的方法没有配置 Merger 合并器(即没有指定 merger 参数),则不会进行结果合并,而是直接将第一个可用的 Invoker 结果返回.下面来看 MergeableClusterInvoker 的具体实现
```
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    String merger = getUrl().getMethodParameter(invocation.getMethodName(), MERGER_KEY);
    // 判断要调用的目标方法是否有合并器，如果没有，则不会进行合并，
    // 找到第一个可用的Invoker直接调用并返回结果
    if (ConfigUtils.isEmpty(merger)) {
        for (final Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                try {
                    return invoker.invoke(invocation);
                } catch (RpcException e) {
                    if (e.isNoInvokerAvailableAfterFilter()) {
                        log.debug("No available provider for service" + getUrl().getServiceKey() + " on group " + invoker.getUrl().getParameter(GROUP_KEY) + ", will continue to try another group.");
                    } else {
                        throw e;
                    }
                }
            }
        }
        return invokers.iterator().next().invoke(invocation);

    }

    // 确定目标方法的返回值类型
    Class<?> returnType;
    try {
        returnType = getInterface().getMethod(
                invocation.getMethodName(), invocation.getParameterTypes()).getReturnType();
    } catch (NoSuchMethodException e) {
        returnType = null;
    }

    // 调用每个Invoker对象(异步方式)，将请求结果记录到results集合中
    Map<String, Result> results = new HashMap<>();
    for (final Invoker<T> invoker : invokers) {
        RpcInvocation subInvocation = new RpcInvocation(invocation, invoker);
        subInvocation.setAttachment(ASYNC_KEY, "true");
        results.put(invoker.getUrl().getServiceKey(), invoker.invoke(subInvocation));
    }

    Object result = null;
    List<Result> resultList = new ArrayList<Result>(results.size());
    // 等待结果返回
    for (Map.Entry<String, Result> entry : results.entrySet()) {
        Result asyncResult = entry.getValue();
        try {
            Result r = asyncResult.get();
            if (r.hasException()) {
                log.error("Invoke " + getGroupDescFromServiceKey(entry.getKey()) +
                                " failed: " + r.getException().getMessage(),
                        r.getException());
            } else {
                resultList.add(r);
            }
        } catch (Exception e) {
            throw new RpcException("Failed to invoke service " + entry.getKey() + ": " + e.getMessage(), e);
        }
    }
    if (resultList.isEmpty()) {
        return AsyncRpcResult.newDefaultAsyncResult(invocation);
    } else if (resultList.size() == 1) {
        return resultList.iterator().next();
    }
    if (returnType == void.class) {
        return AsyncRpcResult.newDefaultAsyncResult(invocation);
    }

    // merger如果以"."开头，后面为方法名，这个方法名是远程目标方法的返回类型中的方法
    // 得到每个Provider节点返回的结果对象之后，会遍历每个返回对象，调用merger参数指定的方法
    if (merger.startsWith(".")) {
        merger = merger.substring(1);
        Method method;
        try {
            method = returnType.getMethod(merger, returnType);
        } catch (NoSuchMethodException e) {
            throw new RpcException("Can not merge result because missing method [ " + merger + " ] in class [ " +
                    returnType.getName() + " ]");
        }
        if (!Modifier.isPublic(method.getModifiers())) {
            method.setAccessible(true);
        }
        // resultList集合保存了所有的返回对象，method是Method对象，也就是merger指定的方法
        // result是最后返回调用方的结果
        result = resultList.remove(0).getValue();
        try {
            if (method.getReturnType() != void.class
                    && method.getReturnType().isAssignableFrom(result.getClass())) {
                for (Result r : resultList) { // 反射调用
                    result = method.invoke(result, r.getValue());
                }
            } else {
                for (Result r : resultList) { // 反射调用
                    method.invoke(result, r.getValue());
                }
            }
        } catch (Exception e) {
            throw new RpcException("Can not merge result: " + e.getMessage(), e);
        }
    } else {
        Merger resultMerger;
        if (ConfigUtils.isDefault(merger)) {
            // merger参数为true或者default，表示使用默认的Merger扩展实现完成合并
            // 在后面课时中会介绍Merger接口
            resultMerger = MergerFactory.getMerger(returnType);
        } else {
            //merger参数指定了Merger的扩展名称，则使用SPI查找对应的Merger扩展实现对象
            resultMerger = ExtensionLoader.getExtensionLoader(Merger.class).getExtension(merger);
        }
        if (resultMerger != null) {
            List<Object> rets = new ArrayList<Object>(resultList.size());
            for (Result r : resultList) {
                rets.add(r.getValue());
            }
            // 执行合并操作
            result = resultMerger.merge(
                    rets.toArray((Object[]) Array.newInstance(returnType, 0)));
        } else {
            throw new RpcException("There is no merger to merge result.");
        }
    }
    return AsyncRpcResult.newDefaultAsyncResult(result, invocation);
}
```



<br><br>
## <span id="jump13">十三. ZoneAwareClusterInvoker</span>

ZoneAwareCluster 这个 Cluster 实现类的扩展名为 zone-aware,在其 doJoin() 方法中创建的是 ZoneAwareClusterInvoker 类型的 Invoker 对象,具体实现如下
```
protected <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
    return new ZoneAwareClusterInvoker<T>(directory);
}
```

在 Dubbo 中使用多个注册中心的架构如下图所示:
[![y4inbQ.png](https://s3.ax1x.com/2021/02/19/y4inbQ.png)](https://imgchr.com/i/y4inbQ)

Consumer 可以使用 ZoneAwareClusterInvoker 先在多个注册中心之间进行选择,选定注册中心之后,再选择 Provider 节点,如下图所示
[![y4itrF.png](https://s3.ax1x.com/2021/02/19/y4itrF.png)](https://imgchr.com/i/y4itrF)

ZoneAwareClusterInvoker 在多注册中心之间进行选择的策略有以下四种
1. 找到preferred 属性为 true 的注册中心,它是优先级最高的注册中心,只有该中心无可用 Provider 节点时,才会回落到其他注册中心
2. 根据请求中的 zone key 做匹配,优先派发到相同 zone 的注册中心
3. 根据权重(也就是注册中心配置的 weight 属性)进行轮询
4. 如果上面的策略都未命中,则选择第一个可用的 Provider 节点

下面来看 ZoneAwareClusterInvoker 的具体实现
```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    // 首先找到preferred属性为true的注册中心，它是优先级最高的注册中心，只有该中心无可用 Provider 节点时，才会回落到其他注册中心
    for (Invoker<T> invoker : invokers) {
        MockClusterInvoker<T> mockClusterInvoker = (MockClusterInvoker<T>) invoker;
        if (mockClusterInvoker.isAvailable() && mockClusterInvoker.getRegistryUrl()
                .getParameter(REGISTRY_KEY + "." + PREFERRED_KEY, false)) {
            return mockClusterInvoker.invoke(invocation);
        }
    }
    // 根据请求中的registry_zone做匹配，优先派发到相同zone的注册中心
    String zone = (String) invocation.getAttachment(REGISTRY_ZONE);
    if (StringUtils.isNotEmpty(zone)) {
        for (Invoker<T> invoker : invokers) {
            MockClusterInvoker<T> mockClusterInvoker = (MockClusterInvoker<T>) invoker;
            if (mockClusterInvoker.isAvailable() && zone.equals(mockClusterInvoker.getRegistryUrl().getParameter(REGISTRY_KEY + "." + ZONE_KEY))) {
                return mockClusterInvoker.invoke(invocation);
            }
        }
        String force = (String) invocation.getAttachment(REGISTRY_ZONE_FORCE);
        if (StringUtils.isNotEmpty(force) && "true".equalsIgnoreCase(force)) {
            throw new IllegalStateException("...");
        }
    }
    // 根据权重（也就是注册中心配置的weight属性）进行轮询
    Invoker<T> balancedInvoker = select(loadbalance, invocation, invokers, null);
    if (balancedInvoker.isAvailable()) {
        return balancedInvoker.invoke(invocation);
    }
    // 选择第一个可用的 Provider 节点
    for (Invoker<T> invoker : invokers) {
        MockClusterInvoker<T> mockClusterInvoker = (MockClusterInvoker<T>) invoker;
        if (mockClusterInvoker.isAvailable()) {
            return mockClusterInvoker.invoke(invocation);
        }
    }
    throw new RpcException("No provider available in " + invokers);
}
```