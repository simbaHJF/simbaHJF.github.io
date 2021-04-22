---
layout:     post
title:      "Sentinel"
date:       2021-04-21 13:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - sentinel

---






## 导航
[一. Sentinel的应用场景](#jump1)
<br>
[二. Sentinel基本概念](#jump2)
<br>
[三. Sentinel基于Qps流控原理](#jump3)
<br>









<br><br>
## <span id="jump1">一. Sentinel的应用场景</span>

* 流量控制
* 熔断降级


<br>
**<font size="4">流量控制</font>** <br>

Sentinel的流量控制基于滑动窗口


<br>
**<font size="5">熔断降级的设计理念</font>** <br>

熔断降级的设计理念上,Sentinel和Hystrix采取了完全不一样的方式.<br>

Hystrix通过线程池的方式,来对依赖(资源)进行隔离.这样做的好处是资源和资源之间做到了最彻底的隔离.缺点是增加了线程切换的成本,还需要预先给各个资源做线程池大小的分配.<br>

Sentinel在熔断降级上采取了两种手段:
* 通过并发线程数进行限制
	> 和资源池隔离的方式不同,Sentinel通过限制资源并发线程的数量,来减少不稳定资源对其他资源的影响.这样不但没有线程切换的损耗,也不需要预先分配线程池的大小.当某个资源出现不稳定的情况,例如响应时间变长,对资源的直接影响就是会造成线程数的逐步堆积.当线程数在特定资源上堆积到一定数量之后,对该资源的新求就会被拒绝.堆积的线程完成任务后才开始继续接收请求.
* 通过响应时间对资源进行限制
	> 除了对并发线程数进行控制以外,Sentinel还可以通过响应时间来快速降级不稳定资源.当依赖的资源出现响应时间过长后,所有对该资源的访问都会被直接拒绝,直到过了指定的时间窗口之后才重新恢复.



<br><br>
## <span id="jump2">二. Sentinel基本概念</span>

<br>
**<font size="4">Resource</font>** <br>

Resource 是 Sentinel 中最重要的一个概念, Sentinel 通过资源来保护具体的业务代码或其他后方服务.


<br>
**<font size="4">Slot</font>** <br>

Slot 是另一个 Sentinel 中非常重要的概念, Sentinel 的工作流程就是围绕着一个个插槽所组成的插槽链来展开的.需要注意的是每个插槽都有自己的职责,他们各司其职完好的配合,通过一定的编排顺序,来达到最终的限流降级的目的.默认的各个插槽之间的顺序是固定的,因为有的插槽需要依赖其他的插槽计算出来的结果才能进行工作.<br>

但是这并不意味着我们只能按照框架的定义来,Sentinel 通过 SlotChainBuilder 作为 SPI 接口,使得 Slot Chain 具备了扩展的能力.我们可以通过实现 SlotsChainBuilder 接口加入自定义的 slot 并自定义编排各个 slot 之间的顺序,从而可以给 Sentinel 添加自定义的功能.<br>

那SlotChain是在哪创建的呢?是在 CtSph.lookProcessChain() 方法中创建的,并且该方法会根据当前请求的资源先去一个静态的HashMap中获取,如果获取不到才会创建,创建后会保存到HashMap中.这就意味着,同一个资源会全局共享一个SlotChain
[![cblX8g.png](https://z3.ax1x.com/2021/04/21/cblX8g.png)](https://imgtu.com/i/cblX8g)


<br>
**<font size="4">Context</font>** <br>

context中维护着当前调用链的元数据


<br>
**<font size="4">Entry</font>** <br>

Entry 是 Sentinel 中用来表示是否通过限流的一个凭证,就像一个token一样.每次执行 SphU.entry() 或 SphO.entry() 都会返回一个 Entry 给调用者,意思就是告诉调用者,如果正确返回了 Entry 给你,那表示你可以正常访问被 Sentinel 保护的后方服务了,否则 Sentinel 会抛出一个BlockException(如果是 SphO.entry() 会返回false),这就表示调用者想要访问的服务被保护了,也就是说调用者本身被限流了


<br>
**<font size="4">Node</font>** <br>

Node 中保存了资源的实时统计数据,例如:passQps,blockQps,rt等实时数据.正是有了这些统计数据后, Sentinel 才能进行限流、降级等一系列的操作.


<br>
**<font size="4">Metric</font>** <br>

Metric 是 Sentinel 中用来进行实时数据统计的度量接口,node就是通过metric来进行数据统计的.而metric本身也并没有统计的能力,他也是通过Window来进行统计的.基于滑动窗口.



<br><br>
## <span id="jump3">三. Sentinel基于Qps流控原理</span>

API调用方式类似如下(可基于注解,这里只分析API,本质一样):
```
Entry entry = null;
try {
    entry = SphU.entry(resourceName, EntryType.IN, 1);
    // Add pass for parameter.
    passFor(param);
} catch (BlockException e) {
    // block.incrementAndGet();
    blockFor(param);
} catch (Exception ex) {
    // biz exception
    ex.printStackTrace();
} finally {
    // total.incrementAndGet();
    if (entry != null) {
        entry.exit(1, param);
    }
}
```

从入口开始:SphU.entry(),这个方法会申请一个Entry,如果能申请成功,则说明没有被限流,否则会抛出BlockException,表示已经被限流了.跟进方法内部调用,进入entryWithPriority方法的调用:
```
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) {
        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
        // so here init the entry only. No rule checking will be done.
        return new CtEntry(resourceWrapper, null, context);
    }
    if (context == null) {
        // Using default context.
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }
    // Global switch is close, no rule checking will do.
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);
    /*
     * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
     * so no rule checking will be done.
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }
    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```

基本流程如下图所示
[![cLkMcV.md.png](https://z3.ax1x.com/2021/04/22/cLkMcV.md.png)](https://imgtu.com/i/cLkMcV)