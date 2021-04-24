---
layout:     post
title:      "Sentinel 指标统计"
date:       2021-04-22 13:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - sentinel

---








## 导航
[一. 源码调用链](#jump1)
<br>
[二. 滑动窗口计算逻辑](#jump2)
<br>
[三. 资源调用线程数统计](#jump3)
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

这里先来看下LeapArray中的关键属性
```
public LeapArray(int sampleCount, int intervalInMs) {
    AssertUtil.isTrue(sampleCount > 0, "bucket count is invalid: " + sampleCount);
    AssertUtil.isTrue(intervalInMs > 0, "total time interval of the sliding window should be positive");
    AssertUtil.isTrue(intervalInMs % sampleCount == 0, "time span needs to be evenly divided");
    this.windowLengthInMs = intervalInMs / sampleCount;
    this.intervalInMs = intervalInMs;
    this.intervalInSecond = intervalInMs / 1000.0;
    this.sampleCount = sampleCount;
    this.array = new AtomicReferenceArray<>(sampleCount);
}
```

* windowLengthInMs表示窗口长度,ms
* intervalInMs表示在多大的时间跨度内进行多个窗口的划分,ms
* intervalInSecond,与intervalInMs对应,s
* sampleCount,一个时间跨度内,窗口个数
* array,一个原子数组,表示时间跨度内,窗口的数组

在默认情况下,时间跨度intervalInMs为1000ms,跨度内窗口个数sampleCount为2.因此默认下,统计的窗口长度为500ms,一个时间跨度内有2个窗口,之所以这么设置是为了避免窗口太小太多,造成统计时不必要的性能浪费.<br>

另外窗口用WindowWrap表示,其内部包裹MetricBuck,用以真整的完成统计逻辑.<br>

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



<br><br>
## <span id="jump2">二. 滑动窗口计算逻辑</span>

获取一个时间点对应的滑动窗口的逻辑封装在currentWindow方法内:
```
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }
    int idx = calculateTimeIdx(timeMillis);
    // Calculate current bucket start time.
    long windowStart = calculateWindowStart(timeMillis);
    /*
     * Get bucket item at given time from the array.
     *
     * (1) Bucket is absent, then just create a new bucket and CAS update to circular array.
     * (2) Bucket is up-to-date, then just return the bucket.
     * (3) Bucket is deprecated, then reset current bucket and clean all deprecated buckets.
     */
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            /*
             *     B0       B1      B2    NULL      B4
             * ||_______|_______|_______|_______|_______||___
             * 200     400     600     800     1000    1200  timestamp
             *                             ^
             *                          time=888
             *            bucket is empty, so create new and update
             *
             * If the old bucket is absent, then we create a new bucket at {@code windowStart},
             * then try to update circular array via a CAS operation. Only one thread can
             * succeed to update, while other threads yield its time slice.
             */
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) {
                // Successfully updated, return the created bucket.
                return window;
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            /*
             *     B0       B1      B2     B3      B4
             * ||_______|_______|_______|_______|_______||___
             * 200     400     600     800     1000    1200  timestamp
             *                             ^
             *                          time=888
             *            startTime of Bucket 3: 800, so it's up-to-date
             *
             * If current {@code windowStart} is equal to the start timestamp of old bucket,
             * that means the time is within the bucket, so directly return the bucket.
             */
            return old;
        } else if (windowStart > old.windowStart()) {
            /*
             *   (old)
             *             B0       B1      B2    NULL      B4
             * |_______||_______|_______|_______|_______|_______||___
             * ...    1200     1400    1600    1800    2000    2200  timestamp
             *                              ^
             *                           time=1676
             *          startTime of Bucket 2: 400, deprecated, should be reset
             *
             * If the start timestamp of old bucket is behind provided time, that means
             * the bucket is deprecated. We have to reset the bucket to current {@code windowStart}.
             * Note that the reset and clean-up operations are hard to be atomic,
             * so we need a update lock to guarantee the correctness of bucket update.
             *
             * The update lock is conditional (tiny scope) and will take effect only when
             * bucket is deprecated, so in most cases it won't lead to performance loss.
             */
            if (updateLock.tryLock()) {
                try {
                    // Successfully get the update lock, now we reset the bucket.
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // Should not go through here, as the provided time is already behind.
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}


private int calculateTimeIdx(/*@Valid*/ long timeMillis) {
    long timeId = timeMillis / windowLengthInMs;
    // Calculate current index so we can map the timestamp to the leap array.
    return (int)(timeId % array.length());
}


protected long calculateWindowStart(/*@Valid*/ long timeMillis) {
    return timeMillis - timeMillis % windowLengthInMs;
}
```

逻辑还是比较简单清晰的,下图描述了的上述过程:
[![cXUTIg.png](https://z3.ax1x.com/2021/04/23/cXUTIg.png)](https://imgtu.com/i/cXUTIg)



<br><br>
## <span id="jump3">三. 资源调用线程数统计</span>

StatisticSlot中调用StatisticNode.increaseThreadNum()与StatisticNode.decreaseThreadNum()完成对调用该资源的当前线程数的增减统计.这里并没用到滑动窗口,通过LongAdder类型的curThreadNum的原子加减完成.<br>

