---
layout:     post
title:      "CountDownLatch"
date:       2021-05-18 16:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---






## 导航
[一. CountDownLatch对AQS同步器的实现](#jump1)
<br>
[二. 流程小结](#jump2)
<br>










<br><br>
## <span id="jump1">一. CountDownLatch对AQS同步器的实现</span>

源码奉上:
```
/**
 * Synchronization control For CountDownLatch.
 * Uses AQS state to represent count.
 */
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    Sync(int count) {
        setState(count);
    }
    int getCount() {
        return getState();
    }
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

下面是对CountDownLatch的初始化构造方法:
```
/**
 * Constructs a {@code CountDownLatch} initialized with the given count.
 *
 * @param count the number of times {@link #countDown} must be invoked
 *        before threads can pass through {@link #await}
 * @throws IllegalArgumentException if {@code count} is negative
 */
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

其阻塞等待await()及资源释放countDown()源码如下:
```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}


public void countDown() {
    sync.releaseShared(1);
}
```



<br><br>
## <span id="jump2">二. 流程小结</span>

1. 创建CountDownLatch时,初始化资源数state,此时state值为一个正数,非零,因此CountDownLatch初始创建后,便处于被锁定状态
2. 某线程(这里将其称之为A),调用await方法阻塞
3. 随着其他线程对CountDownLatch的countDown方法的调用,会逐步释放资源
4. 当state数释放为零时,唤醒同步队列中的head的后继node,也即前面调用await阻塞住的A线程
5. A线程获得锁,完成后续执行