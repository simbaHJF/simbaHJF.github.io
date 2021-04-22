---
layout:     post
title:      "Sentinel 滑动窗口"
date:       2021-04-22 13:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - sentinel

---








## 导航
[一. 源码调用链](#jump1)
<br>










<br><br>
## <span id="jump1">一. 源码调用链</span>

在Sentinel的slotChain中有一个非常重要的slot,即StatisticSlot,它负责对各维度的调用指标信息进行统计,后续的流量控制和熔断降级,都是基于StatisticSlot中的统计信息结合配置的Rule规则进行相应判断来实现的.<br>


<br>
**<font size="4">StatisticSlot</font>** <br>

```
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    try {
        // Do some checking.
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
        // Request passed, add thread count and pass count.
        node.increaseThreadNum();
        node.addPassRequest(count);
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }
        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }
        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (PriorityWaitException ex) {
        node.increaseThreadNum();
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
        }
        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
        }
        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setBlockError(e);
        // Add block count.
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }
        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseBlockQps(count);
        }
        // Handle block event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onBlocked(e, context, resourceWrapper, node, count, args);
        }
        throw e;
    } catch (Throwable e) {
        // Unexpected internal error, set error to current entry.
        context.getCurEntry().setError(e);
        throw e;
    }
}
```

大体流程为:
1. 通过node中的当前的实时统计指标信息进行规则校验
2. 如果通过了校验,则重新更新node中的实时指标数据
3. 如果被block或出现了异常了,则重新更新node中block的指标或异常指标

从上面的代码中可以很清晰的看到,所有的实时指标的统计都是在node中进行的.这里我们拿qps的指标进行分析,看sentinel是怎么统计出qps的,这里可以事先透露下他是通过滑动时间窗口来统计的,而滑动窗口就是本篇文章的重点<br>


<br>
**<font size="4">DefaultNode和ClusterNode</font>** <br>

node.addPassRequest() 这段代码是在fireEntry执行之后执行的,这意味着,当前请求通过了sentinel的流控等规则,此时需要将当次请求记录下来,也就是执行 node.addPassRequest() 这行代码,现在我们进入这个代码看看.具体的代码如下所示

```
public void addPassRequest(int count) {
    super.addPassRequest(count);
    this.clusterNode.addPassRequest(count);
}
```

首先我们知道这里的node是一个 DefaultNode 实例,这里特别补充一个 DefaultNode 和 ClusterNode 的区别:
* DefaultNode:保存着某个resource在某个context中的实时指标,每个DefaultNode都指向一个ClusterNode
* ClusterNode:保存着某个resource在所有的context中实时指标的总和,同样的resource会共享同一个ClusterNode,不管他在哪个context中


<br>
**<font size="4">StatisticNode</font>** <br>

进入 addPassRequest 方法内部:
```
private transient Metric rollingCounterInSecond = new ArrayMetric(1000 / SampleCountProperty.sampleCount, IntervalProperty.INTERVAL);
 
private transient Metric rollingCounterInMinute = new ArrayMetric(1000, 2 * 60);
 
@Override
public void addPassRequest() {
    rollingCounterInSecond.addPass();
    rollingCounterInMinute.addPass();
}
```


<br>
**<font size="4">Metric</font>** <br>

具体的增加pass指标是通过一个叫 Metric 的接口进行操作的,并且是通过 ArrayMetric 这种实现类,现在我们在进入 ArrayMetric 中看一下.具体的代码如下所示:
```
private final LeapArray<MetricBucket> data;

public ArrayMetric(int sampleCount, int intervalInMs) {
    this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
}
 
@Override
public void addPass(int count) {
    WindowWrap<MetricBucket> wrap = data.currentWindow();
    wrap.value().addPass(count);
}
```


<br>
**<font size="4">LeapArray和MetricBucket</font>** <br>

继续跟进代码,发现 wrap.value().addPass() 是执行的 wrap 对象所包装的 Window 对象的 addPass 方法,这里就是最终的增加qps中q的值的地方了.进入 MetricBucket 类中看一下,具体的代码如下
```
public void addPass(int n) {
    add(MetricEvent.PASS, n);
}

public MetricBucket add(MetricEvent event, long n) {
    counters[event.ordinal()].add(n);
    return this;
}
```

对counters数组初始化的代码如下:
```
public MetricBucket() {
    MetricEvent[] events = MetricEvent.values();
    this.counters = new LongAdder[events.length];
    for (MetricEvent event : events) {
        counters[event.ordinal()] = new LongAdder();
    }
    initMinRt();
}
```

是通过 LongAdder 来保存各种指标的值的,看到 LongAdder 是不是立刻就想到 AtomicLong 了? 但是这里为什么不用 AtomicLong, 而是用 LongAdder 呢? 主要是 LongAdder 在高并发下有更好的吞吐量,代价是花费了更多的空间,典型的以空间换时间<br>

LongAdder,一方面避免伪共享,另一方面CAS时进行锁分段,两方面提高并发性能.<br>

[![cLcY9O.png](https://z3.ax1x.com/2021/04/22/cLcY9O.png)](https://imgtu.com/i/cLcY9O)