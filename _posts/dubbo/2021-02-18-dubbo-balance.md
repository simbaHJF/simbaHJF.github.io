---
layout:     post
title:      "Dubbo 负载均衡机制"
date:       2021-02-18 13:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---


## 导航
[一. 负载均衡简介](#jump1)
<br>
[二. ConditionRouterFactory&ConditionRouter](#jump2)
<br>
[三. ConsistentHashLoadBalance](#jump3)
<br>




<br><br>
## <span id="jump1">一. 负载均衡简介</span>

[![yRR9tU.png](https://s3.ax1x.com/2021/02/18/yRR9tU.png)](https://imgchr.com/i/yRR9tU)

LoadBalance(负载均衡)的职责是将网络请求或者其他形式的负载"均摊"到不同的服务节点上,从而避免服务集群中部分节点压力过大、资源紧张,而另一部分节点比较空闲的情况.<br>

通过合理的负载均衡算法,我们希望可以让每个服务节点获取到适合自己处理能力的负载,实现处理能力和流量的合理分配.常用的负载均衡可分为软件负载均衡(比如,日常工作中使用的 Nginx)和硬件负载均衡(主要有 F5、Array、NetScaler 等,不过开发工程师在实践中很少直接接触到).<br>

常见的 RPC 框架中都有负载均衡的概念和相应的实现,Dubbo 也不例外.Dubbo 需要对 Consumer 的调用请求进行分配,避免少数 Provider 节点负载过大,而剩余的其他 Provider 节点处于空闲的状态.因为当 Provider 负载过大时,就会导致一部分请求超时、丢失等一系列问题发生,造成线上故障.<br>

Dubbo 提供了 5 种负载均衡实现,分别是:
* 基于 Hash 一致性的 ConsistentHashLoadBalance
* 基于权重随机算法的 RandomLoadBalance
* 基于最少活跃调用数算法的 LeastActiveLoadBalance
* 基于加权轮询算法的 RoundRobinLoadBalance
* 基于最短响应时间的 ShortestResponseLoadBalance



<br><br>
## <span id="jump2">二. LoadBalance 接口</span>

Dubbo 提供的负载均衡实现,都是 LoadBalance 接口的实现类,如下图所示
[![yRWJxJ.png](https://s3.ax1x.com/2021/02/18/yRWJxJ.png)](https://imgchr.com/i/yRWJxJ)

LoadBalance 是一个扩展接口,默认使用的扩展实现是 RandomLoadBalance,其定义如下所示,其中的 @Adaptive 注解参数为 loadbalance,即动态生成的适配器会按照 URL 中的 loadbalance 参数值选择扩展实现类.
```
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

LoadBalance 接口中 select() 方法的核心功能是根据传入的 URL 和 Invocation,以及自身的负载均衡算法,从 Invoker 集合中选择一个 Invoker 返回.<br>

AbstractLoadBalance 抽象类并没有真正实现 select() 方法,只是对 Invoker 集合为空或是只包含一个 Invoker 对象的特殊情况进行了处理,具体实现如下:
```
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    if (CollectionUtils.isEmpty(invokers)) { 
        return null; // Invoker集合为空，直接返回null
    }
    if (invokers.size() == 1) { // Invoker集合只包含一个Invoker，则直接返回该Invoker对象
        return invokers.get(0);
    }
    // Invoker集合包含多个Invoker对象时，交给doSelect()方法处理，这是个抽象方法，留给子类具体实现
    return doSelect(invokers, url, invocation);
}
```

另外,AbstractLoadBalance 还提供了一个 getWeight() 方法,该方法用于计算 Provider 权重,具体实现如下:
```
int getWeight(Invoker<?> invoker, Invocation invocation) {
    int weight;
    URL url = invoker.getUrl();
    if (REGISTRY_SERVICE_REFERENCE_PATH.equals(url.getServiceInterface())) {
        // 如果是RegistryService接口的话，直接获取权重即可
        weight = url.getParameter(REGISTRY_KEY + "." + WEIGHT_KEY, DEFAULT_WEIGHT);
    } else {
        weight = url.getMethodParameter(invocation.getMethodName(), WEIGHT_KEY, DEFAULT_WEIGHT);
        if (weight > 0) {
            // 获取服务提供者的启动时间戳
            long timestamp = invoker.getUrl().getParameter(TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
                // 计算Provider运行时长
                long uptime = System.currentTimeMillis() - timestamp;
                if (uptime < 0) {
                    return 1;
                }
                // 计算Provider预热时长
                int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
                // 如果Provider运行时间小于预热时间，则该Provider节点可能还在预热阶段，需要重新计算服务权重(降低其权重)
                if (uptime > 0 && uptime < warmup) {
                    weight = calculateWarmupWeight((int)uptime, warmup, weight);
                }
            }
        }
    }
    return Math.max(weight, 0);
}
```

calculateWarmupWeight() 方法的目的是对还在预热状态的 Provider 节点进行降权,避免 Provider 一启动就有大量请求涌进来.服务预热是一个优化手段,这是由 JVM 本身的一些特性决定的,例如,JIT 等方面的优化,我们一般会在服务启动之后,让其在小流量状态下运行一段时间,然后再逐步放大流量.
```
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    // 计算权重，随着服务运行时间uptime增大，权重ww的值会慢慢接近配置值weight
    int ww = (int) ( uptime / ((float) warmup / weight));
    return ww < 1 ? 1 : (Math.min(ww, weight));
}
```

了解了 LoadBalance 接口的定义以及 AbstractLoadBalance 提供的公共能力之后,下面我们开始逐个介绍 LoadBalance 接口的具体实现.<br>



<br><br>
## <span id="jump3">三. ConsistentHashLoadBalance</span>

ConsistentHashLoadBalance 底层使用一致性 Hash 算法实现负载均衡.这里先来简单介绍一下一致性 Hash 算法相关的知识点.<br>

**<font size="5">一致性 Hash 简析</font>** <br>

一致性 Hash 负载均衡可以让参数相同的请求每次都路由到相同的服务节点上,这种负载均衡策略可以在某些 Provider 节点下线的时候,让这些节点上的流量平摊到其他 Provider 上,不会引起流量的剧烈波动.<br>

下面我们通过一个示例,简单介绍一致性 Hash 算法的原理.<br>

假设现在有 1、2、3 三个 Provider 节点对外提供服务,有 100 个请求同时到达,如果想让请求尽可能均匀地分布到这三个 Provider 节点上,我们可能想到的最简单的方法就是 Hash 取模,即 hash(请求参数) % 3.如果参与 Hash 计算的是请求的全部参数,那么参数相同的请求将会落到同一个 Provider 节点上.不过此时如果突然有一个 Provider 节点出现宕机的情况,那我们就需要对 2 取模,即请求会重新分配到相应的 Provider 之上.在极端情况下,甚至会出现所有请求的处理节点都发生了变化,这就会造成比较大的波动.<br>

为了避免因一个 Provider 节点宕机,而导致大量请求的处理节点发生变化的情况,我们可以考虑使用一致性 Hash 算法.一致性 Hash 算法的原理也是取模算法,与 Hash 取模的不同之处在于:Hash 取模是对 Provider 节点数量取模,而一致性 Hash 算法是对 2^32 取模.<br>

一致性 Hash 算法需要同时对 Provider 地址以及请求参数进行取模
```
hash(Provider地址) % 2^32
hash(请求参数) % 2^32
```

Provider 地址和请求经过对 2^32 取模得到的结果值,都会落到一个 Hash 环上,如下图所示
[![yRhyDA.png](https://s3.ax1x.com/2021/02/18/yRhyDA.png)](https://imgchr.com/i/yRhyDA)

我们按顺时针的方,依次将请求分发到对应的 Provider.这样,当某台 Provider 节点宕机或增加新的 Provider 节点时,只会影响这个 Provider 节点对应的请求.<br>

在理想情况下,一致性 Hash 算法会将这三个 Provider 节点均匀地分布到 Hash 环上,请求也可以均匀地分发给这三个 Provider 节点.但在实际情况中,这三个 Provider 节点地址取模之后的值,可能差距不大,这样会导致大量的请求落到一个 Provider 节点上,如下图所示.
[![yR48G8.png](https://s3.ax1x.com/2021/02/18/yR48G8.png)](https://imgchr.com/i/yR48G8)

这就出现了数据倾斜的问题.所谓数据倾斜是指由于节点不够分散,导致大量请求落到了同一个节点上,而其他节点只会接收到少量请求的情况.<br>

为了解决一致性 Hash 算法中出现的数据倾斜问题,又演化出了 Hash 槽的概念.<br>

Hash 槽解决数据倾斜的思路是:既然问题是由 Provider 节点在 Hash 环上分布不均匀造成的,那么可以虚拟出 n 组 P1、P2、P3 的 Provider 节点,让多组 Provider 节点相对均匀地分布在 Hash 环上.如下图所示,相同阴影的节点均为同一个 Provider 节点,比如 P1-1、P1-2……P1-99 表示的都是 P1 这个 Provider 节点.引入 Provider 虚拟节点之后,让 Provider 在圆环上分散开来,以避免数据倾斜问题.
[![yR4qLd.png](https://s3.ax1x.com/2021/02/18/yR4qLd.png)](https://imgchr.com/i/yR4qLd)

**<font size="5">ConsistentHashSelector 实现分析</font>** <br>