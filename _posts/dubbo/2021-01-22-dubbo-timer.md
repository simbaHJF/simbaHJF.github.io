---
layout:     post
title:      "时间轮"
date:       2021-01-22 23:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---

## 导航
[一. 为什么需要时间轮](#jump1)
<br>
[二. 什么是时间轮](#jump2)
<br>
[三. 核心接口](#jump3)
<br>
[四. 核心源码分析](#jump4)
<br>
[五. 一点思考](#jump5)




<br><br>
## <span id="jump1">一. 为什么需要时间轮</span>

JDK 提供的 java.util.Timer 和 DelayedQueue 等工具类,可以帮助我们实现简单的定时任务管理,其底层实现使用的是堆这种数据结构,存取操作的复杂度都是 O(nlog(n)),无法支持大量的定时任务.在定时任务量比较大、性能要求比较高的场景中,为了将定时任务的存取操作以及取消操作的时间复杂度降为 O(1),一般会使用时间轮的方式.


<br><br>
## <span id="jump2">二. 什么是时间轮</span>

时间轮是一种高效的、批量管理定时任务的调度模型.时间轮一般会实现成一个环形结构,类似一个时钟,分为很多槽,一个槽代表一个时间间隔,每个槽使用双向链表存储定时任务;指针周期性地跳动,跳动到一个槽位,就执行该槽位的定时任务.<br>

[![so4yss.png](https://s3.ax1x.com/2021/01/23/so4yss.png)](https://imgchr.com/i/so4yss)

需要注意的是,单层时间轮的容量和精度都是有限的,对于精度要求特别高、时间跨度特别大或是海量定时任务需要调度的场景,通常会使用多级时间轮以及持久化存储与时间轮结合的方案.<br>

<font color="red">或者采用在时间轮中加入周期的概念来解决.</font>

<br>
在 Dubbo 中,时间轮的实现位于 dubbo-common 模块的org.apache.dubbo.common.timer 包中.<br>


<br><br>
## <span id="jump3">三. 核心接口</span>

**<font size="3">TimerTask,Timeout,Timer</font>**

在 Dubbo 中,所有的定时任务都要继承 TimerTask 接口,TimerTask 接口非常简单,只定义了一个 run() 方法,代表要执行的任务逻辑,该方法的入参是一个 Timeout 接口的对象.Timeout 对象与 TimerTask 对象一一对应,两者的关系类似于线程池返回的 Future 对象与提交到线程池中的任务对象之间的关系.通过 Timeout 对象,我们不仅可以查看定时任务的状态,还可以操作定时任务(例如取消关联的定时任务).Timeout 接口中的方法如下图所示:
[![soIeN6.png](https://s3.ax1x.com/2021/01/23/soIeN6.png)](https://imgchr.com/i/soIeN6)

Timer 接口定义了定时器的基本行为,如下图所示,其核心是 newTimeout() 方法:提交一个定时任务(TimerTask)并返回关联的 Timeout 对象,这有点类似于向线程池提交任务的感觉
[![sThX6J.png](https://s3.ax1x.com/2021/01/23/sThX6J.png)](https://imgchr.com/i/sThX6J)



**<font size="3">HashedWheelTimeout</font>**

HashedWheelTimeout 是 Timeout 接口的唯一实现,是 HashedWheelTimer 的内部类.HashedWheelTimeout 扮演了两个角色:
* 第一个, 时间轮中双向链表的节点,即定时任务TimerTask在HashedWheelTimer中的容器.
* 第二个,定时任务TimerTask提交到HashedWheelTimer之后的句柄(Handle),用于在时间轮外部查看和控制定时任务.

HashedWheelTimeout中的核心字段如下:
* prev、next(HashedWheelTimeout类型),分别对应当前定时任务在链表中的前驱结点和后继节点.
* task(TimerTask类型),指实际被调度的任务
* deadline(long类型),指定时任务执行的时间.这个时间是在创建HashedWheelTimeout时指定的,计算公式是:<br>
``
currentTime（创建 HashedWheelTimeout 的时间） + delay（任务延迟时间） - startTime（HashedWheelTimer 的启动时间），时间单位为纳秒
``

* state(volatile类型),只定时任务当前所处状态,可选的有三个,分别是INIT(0),CANCELLED(1)和EXPIRED(2).另外,还有一个STATE_UPDATER字段(AtomicIntegerFieldUpdater类型)实现state状态变更的原子性.
* remainingRounds(long类型),指当前任务剩余的时钟周期数.时间轮所能表示的时间长度是有限的,在任务到期时间与当前时刻的时间差超过时间轮单圈能表示的时长的情况下,就出现了套圈的情况,需要该字段值表示剩余的时钟周期.

HashedWheelTimeout中的核心方法有:
* isCancelled(),isExpired(),state()方法,主要用于检查当前HashedWheelTimeout状态.
* cancel()方法,将当前HashedWheelTimeout的状态设置为CANCELLED,并将当前HashedWheelTimeout添加到cancelledTimeouts队列中等待销毁.
* expire()方法,将当前HashedWheelTimeout从时间轮中删除.


**<font size="3">HashedWheelBucket</font>**

HashedWheelBucket是时间轮中的一个槽,时间轮中的槽实际上就是一个用于缓存和管理双向链表的容器,双向链表中的每一个节点都是HashedWheelTimeout对象,也就关联了一个TimerTask定时任务.<br>

HashedWheelBucket持有双向链表的首尾两个节点,分别是head和tail两个字段,再加上每个HashedWheelTimeout节点均持有前驱和后继的引用,这样就可以正向或是逆向遍历整个双向链表了.<br>

下面来看HashedWheelBucket中的核心方法
* addTimeout()方法:新增 HashedWheelTimeout 到双向链表的尾部
* pollTimeout()方法:移除双向链表中的头结点,并将其返回
* remove()方法:从双向链表中移除指定的HashedWheelTimeout节点
* clearTimeouts()方法:循环调用 pollTimeout()方法处理整个双向链表,并返回所有未超时或者未被取消的任务
* expireTimeouts()方法:遍历双向链表中的全部 HashedWheelTimeout 节点. 在处理到期的定时任务时,会通过 remove() 方法取出,并调用其 expire() 方法执行;对于已取消的任务,通过 remove() 方法取出后直接丢弃;对于未到期的任务,会将 remainingRounds 字段（剩余时钟周期数）减一


**<font size="3">HashedWheelTimer</font>**

HashedWheelTimer 是 Timer 接口的实现, 它通过时间轮算法实现了一个定时器.HashedWheelTimer 会根据当前时间轮指针选定对应的槽(HashedWheelBucket),从双向链表的头部开始迭代,对每个定时任务(HashedWheelTimeout)进行计算,属于当前时钟周期则取出运行,不属于则将其剩余的时钟周期数减一操作.<br>

下面来看下HashedWheelTimer的核心属性.
* workerState(volatile int类型):时间轮当前所处状态,可选值有init,started,shutdown.同时,有相应的AtomicIntegerFieldUpdater实现workerState的原子修改.
* startTime(long类型):当前时间轮的启动时间,提交到该时间轮的定时任务的deadline字段值均以该时间戳为起点进行计算.
* wheel(HashedWheelBucket[]类型):该数组就是时间轮的环形队列,每一个元素都是一个槽.当指定时间轮槽数为n时,实际上会取大于且最靠近n的2的幂方值.
* timeouts,cancelledTimeouts(LinkedBlockingQueue类型):timeouts队列用于缓冲外部提交到时间轮中的定时任务,cancelledTimeouts队列用于暂存取消的定时任务.HashedWheelTimer会在处理HashedWheelBucket的双向链表之前,先处理这两个队列中的数据.
* tick(long类型):该字段在HashedWheelTimer$Worker中,是时间轮的指针,是一个步长为1的单调递增计数器.
* mask(int类型):掩码,mask = wheel.length - 1,执行ticks & mask便能定位到对应的时钟槽.
* tickDuration(long类型):时间指针每次加1所代表的实际时间,或者说时间轮上过一格代表的实际时间,单位为纳秒
* pendingTimeouts(AtomicLong类型):当前时间轮剩余的定时任务总数
* workerThread(Thread类型):时间轮内部真正执行定时任务的线程
* worker(Worker类型):实现Runable接口,真正执行定时任务的逻辑封装在这个Runable对象中


时间轮对外提供了一个 newTimeout() 接口用于提交定时任务,在定时任务进入到 timeouts 队列之前会先调用 start() 方法启动时间轮,其中会完成下面两个关键步骤:
1. 确定时间轮的 startTime 字段
2. 启动 workerThread 线程,开始执行 worker 任务

之后根据 startTime 计算该定时任务的 deadline 字段,最后才能将定时任务封装成 HashedWheelTimeout 并添加到 timeouts 队列.<br>

时间轮指针一次转动的全流程如下:
1. 时间轮指针转动,时间轮周期开始
2. 清理用户主动取消的定时任务,这些定时任务在用户取消时,会记录到cancelledTimeouts队列中.每次指针转动的时候,时间轮都会清理该队列.
3. 将缓存在timeouts队列中的定时任务转移到时间轮中对应的槽中.
4. 根据当前指针定位对应槽,处理该槽位的双向链表中的定时任务
5. 检测时间轮的状态.如果时间轮处于运行状态,则循环执行上述步骤,不断执行定时任务.如果时间轮处于停止状态,则执行下面的步骤获取到未被执行的定时任务并加入 unprocessedTimeouts 队列:遍历时间轮中每个槽位,并调用 clearTimeouts() 方法;对 timeouts 队列中未被加入槽中循环调用 poll()
6. 最后再次清理cancelledTimeouts队列中用户主动取消的定时任务



<br><br>
## <span id="jump4">四. 核心源码分析</span>

下面来看下HashedWheelTimer核心代码:
```
@Override
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
        pendingTimeouts.decrementAndGet();
        throw new RejectedExecutionException("Number of pending timeouts ("
                + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                + "timeouts (" + maxPendingTimeouts + ")");
    }
    start();
    // Add the timeout to the timeout queue which will be processed on the next tick.
    // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    // Guard against overflow.
    if (delay > 0 && deadline < 0) {
        deadline = Long.MAX_VALUE;
    }
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    timeouts.add(timeout);
    return timeout;
}
```

跟进start()方法内部看下具体实现:
```
public void start() {
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }
    // Wait until the startTime is initialized by the worker.
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```

这了是一个状态模式的应用,通过WORKER_STATE_UPDATER原子性的更新时间轮状态workerState字段.在并发情况下,会阻塞在startTimeInitialized.await()处,等待workerThread启动完成.这里startTimeInitialized是一个CountDownLatch(1)的同步器.<br>


再来看下workerThread线程中执行的工作,也就是内部类Worker的实现:
```
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();
    private long tick;
    @Override
    public void run() {
        // Initialize the startTime.
        startTime = System.nanoTime();
        if (startTime == 0) {
            // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
            startTime = 1;
        }
        // Notify the other threads waiting for the initialization at start().
        startTimeInitialized.countDown();
        do {
            final long deadline = waitForNextTick();
            if (deadline > 0) {
                int idx = (int) (tick & mask);
                processCancelledTasks();
                HashedWheelBucket bucket =
                        wheel[idx];
                transferTimeoutsToBuckets();
                bucket.expireTimeouts(deadline);
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
        // Fill the unprocessedTimeouts so we can return them from stop() method.
        for (HashedWheelBucket bucket : wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        for (; ; ) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        processCancelledTasks();
    }
    private void transferTimeoutsToBuckets() {
        // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
        // adds new timeouts in a loop.
        for (int i = 0; i < 100000; i++) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                // Was cancelled in the meantime.
                continue;
            }
            long calculated = timeout.deadline / tickDuration;
            timeout.remainingRounds = (calculated - tick) / wheel.length;
            // Ensure we don't schedule for past.
            final long ticks = Math.max(calculated, tick);
            int stopIndex = (int) (ticks & mask);
            HashedWheelBucket bucket = wheel[stopIndex];
            bucket.addTimeout(timeout);
        }
    }
    private void processCancelledTasks() {
        for (; ; ) {
            HashedWheelTimeout timeout = cancelledTimeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            try {
                timeout.remove();
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown while process a cancellation task", t);
                }
            }
        }
    }
    /**
     * calculate goal nanoTime from startTime and current tick number,
     * then wait until that goal has been reached.
     *
     * @return Long.MIN_VALUE if received a shutdown request,
     * current time otherwise (with Long.MIN_VALUE changed by +1)
     */
    private long waitForNextTick() {
        long deadline = tickDuration * (tick + 1);
        for (; ; ) {
            final long currentTime = System.nanoTime() - startTime;
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;
            if (sleepTimeMs <= 0) {
                if (currentTime == Long.MIN_VALUE) {
                    return -Long.MAX_VALUE;
                } else {
                    return currentTime;
                }
            }
            if (isWindows()) {
                sleepTimeMs = sleepTimeMs / 10 * 10;
            }
            try {
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }
    Set<Timeout> unprocessedTimeouts() {
        return Collections.unmodifiableSet(unprocessedTimeouts);
    }
}
```



<br><br>
## <span id="jump5">五. 一点思考</span>

向HashedWheelTimer添加定时任务时为什么采用先添加到timeouts队列中,再由worker将其添加到时间轮中呢?为什么不直接放进时间轮里?<br>

我觉得这里是因为对于一个时间轮资源,多个线程并发向时间轮中某个桶管理的双向链表中直接增改节点会造成并发问题,如果为了解决此问题而引入一些同步机制,又会影响性能.而HashedWheelTimer设计的方式,通过一个缓冲队列timeouts和worker配合,屏蔽了这种并发问题,降低了复杂性.<br>

<font color="red">另外一点需要注意,如果加入时间轮中的任务在执行时不是放到一个业务线程池中进行处理,而是在worker时间轮worker线程中执行的话,如果是长耗时任务,会造成后面任务的阻塞与等待.</font>