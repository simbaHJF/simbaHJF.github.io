---
layout:     post
title:      "ThreadPoolExecutor"
date:       2021-05-25 15:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---





## 导航
[一. 线程池状态](#jump1)
<br>
[二. 线程池执行流程](#jump2)
<br>










<br><br>
## <span id="jump1">一. 线程池状态</span>

<br>
**<font size="4">核心属性:</font>** <br>

```
//用来标记线程池状态（高3位），线程个数（低29位）
//默认是RUNNING状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//线程个数掩码位数，并不是所有平台int类型是32位，所以准确说是具体平台下Integer的二进制位数-3后的剩余位数才是线程的个数
private static final int COUNT_BITS = Integer.SIZE - 3;

//线程最大个数(低29位)00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;



//（高3位）：11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;

//（高3位）：00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;

//（高3位）：00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;

//（高3位）：01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;

//（高3位）：01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 获取高三位 运行状态
private static int runStateOf(int c)     { return c & \~CAPACITY; }

//获取低29位 线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }

//计算ctl新值，线程状态 与 线程个数
private static int ctlOf(int rs, int wc) { return rs | wc; }
```


<br>
**<font size="4">线程池状态</font>** <br>

* RUNNING : 接受新任务并处理阻塞队列里的任务
* SHUTDOWN : 拒绝新任务但是处理阻塞队列里的任务
	> 调用线程池的shutdown()方法时,线程池由RUNNING -> SHUTDOWN
* STOP : 拒绝新任务并且抛弃阻塞队列里的任务,同时会中断正在处理的任务
	> 调用线程池的shutdownNow()方法时,线程池由(RUNNING or SHUTDOWN ) -> STOP
* TIDYING : 当所有的任务已终止,ctl记录的"任务数量"为0,线程池会变为TIDYING状态.
	> 当线程池变为TIDYING状态时,会执行钩子函数terminated(). terminated()在ThreadPoolExecutor类中是空的,若用户想在线程池变为TIDYING时,进行相应的处理;可以通过重载terminated()函数来实现<br>
	  当线程池在SHUTDOWN状态下,阻塞队列为空并且线程池中执行的任务也为空时,就会由 SHUTDOWN -> TIDYING <br>
	  当线程池在STOP状态下,线程池中执行的任务为空时,就会由STOP -> TIDYING
* TERMINATED : 终止状态,terminated方法调用完成以后的状态

[![2pBAwq.png](https://z3.ax1x.com/2021/05/26/2pBAwq.png)](https://imgtu.com/i/2pBAwq)


<br>
**<font size="4">线程池构造参数</font>** <br>

* corePoolSize : 线程池核心线程个数
* maximunPoolSize : 线程池最大线程数量
* workQueue : 用于保存等待执行的任务的阻塞队列
* ThreadFactory : 创建线程的工厂
* RejectedExecutionHandler : 拒绝策略
* keeyAliveTime : 存活时间.如果当前线程池中的线程数量比核心线程数量要多,并且是闲置状态的话,这些闲置的线程能存活的最大时间
* TimeUnit : 存活时间的时间单位


拒绝策略: 
* AbortPolicy : 这是线程池默认的拒绝策略,在任务不能再提交的时候,抛出异常,及时反馈程序运行状态
* DiscardPolicy : 丢弃任务,但是不抛出异常,后续提交的任务都会被丢弃,且是静默丢弃
* DiscardOldestPolicy : 丢弃队列最前面的任务,然后重新提交被拒绝的任务
* CallerRunsPolicy : 如果任务添加失败,那么主线程就会自己调用执行器中的 executor 方法来执行该任务



<br><br>
## <span id="jump2">二. 线程池执行流程</span>

[![2ps9SO.png](https://z3.ax1x.com/2021/05/26/2ps9SO.png)](https://imgtu.com/i/2ps9SO)



<br><br>
## <span id="jump3">三. 几种常用阻塞队列分析</span>

<br>
**<font size="4">LinkedBlockingQueue</font>** <br>

由内部Node节点组成的单向链表形式阻塞队列,并发安全性由其内部的putLock(ReentrantLock类型)与takeLock(ReentrantLock类型)保证,put时需要获取putLock锁,take时需要获取takeLock锁<br>

默认无界,事实上其实并不是,而是将capacity长度设置为了Integer.MAX_VALUE,感官上像是无界.<br>

take时队列为空则阻塞,通过notEmpty(Condition类型)来进行控制.<br>

put时队列满则阻塞,通过notFull(Condition类型)来进行控制.<br>

<font color="red">因为put和take是两把锁,所以put和take可以并发</font> <br>


<br>
**<font size="4">ArrayBlockingQueue</font>** <br>

底层由数组支撑的有界队列,通过putIndex,takeIndex和count字段配合,将数组用作循环列表使用.并发安全性由内部的lock(ReentrantLock类型)保障,put及take元素时都需要先获取该锁.<br>

take时队列为空则阻塞,通过notEmpty(Condition类型)来进行控制.<br>

put时队列满则阻塞,通过notFull(Condition类型)来进行控制.<br>

<font color="red">注意,这里只有一把锁,因此take和put两种操作无法并发,因此并发性能不及LinkedBlockingQueue</font> <br>


<br>
**<font size="4">SynchronousQueue</font>** <br>

SynchronousQueue 是一个没有数据缓冲的BlockingQueue,它内部没有容器,一个生产线程,当它生产产品(即put 或offer 的时候),如果当前没有人想要消费产品(即当前没有线程执行take),此时生产线程会有两种情况
*  阻塞,等待一个消费线程调用take操作,take操作将会唤醒该生产线程,同时消费线程会获取生产线程的产品(即数据传递),这样的一个过程称为一次配对过程(当然也可以先take后put,原理是一样的)
*  立即返回失败

来看下源码
```
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}

E transfer(E e, boolean timed, long nanos) {
    /* Basic algorithm is to loop trying to take either of
     * two actions:
     *
     * 1. If queue apparently empty or holding same-mode nodes,
     *    try to add node to queue of waiters, wait to be
     *    fulfilled (or cancelled) and return matching item.
     *
     * 2. If queue apparently contains waiting items, and this
     *    call is of complementary mode, try to fulfill by CAS'ing
     *    item field of waiting node and dequeuing it, and then
     *    returning matching item.
     *
     * In each case, along the way, check for and try to help
     * advance head and tail on behalf of other stalled/slow
     * threads.
     *
     * The loop starts off with a null check guarding against
     * seeing uninitialized head or tail values. This never
     * happens in current SynchronousQueue, but could if
     * callers held non-volatile/final ref to the
     * transferer. The check is here anyway because it places
     * null checks at top of loop, which is usually faster
     * than having them implicitly interspersed.
     */
    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);
    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
            if (t != tail)                  // inconsistent read
                continue;
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
            // 1. 此处条件会按是否允许超时和超时时长两个参数判断是否立即返回null,代表失败
            if (timed && nanos <= 0)        // can't wait
                return null;
            if (s == null)
                s = new QNode(e, isData);
            if (!t.casNext(null, s))        // failed to link in
                continue;
            advanceTail(t, s);              // swing tail and wait
            // 2. 此处会阻塞
            Object x = awaitFulfill(s, e, timed, nanos);
            if (x == s) {                   // wait was cancelled
                clean(t, s);
                return null;
            }
            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;
        } else {                            // complementary-mode
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read
            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // m cancelled
                !m.casItem(x, e)) {         // lost CAS
                advanceHead(h, m);          // dequeue and retry
                continue;
            }
            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```

由1,2两点可以看出其两种返回方式.再结合线程池源码,当没有空闲worker线程来处理提交的任务时,它会从workQueue.offer(command)处失败,而当无法再增加worker线程数的时候,它会从reject方法返回,也就是走拒绝策略
```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

Dubbo中默认下采用的是FixedThreadPool,该线程池中在Dubbo默认参数下,采用的就是SynchronousQueue队列,当dubbo的工作线程全都没有空闲时,其就会将请求立即拒绝,也就是常见的Dubbo线程打满的现象,之后该请求可通过失败重试再次尝试请求.

[![2itkxH.png](https://z3.ax1x.com/2021/05/27/2itkxH.png)](https://imgtu.com/i/2itkxH)