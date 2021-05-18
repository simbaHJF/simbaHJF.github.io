---
layout:     post
title:      "ReentrantLock"
date:       2021-05-18 15:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---







## 导航
[一. ReentrantLock中自定义AQS同步器的继承关系](#jump1)
<br>
[二. 公平与非公平锁的实现原理](#jump2)
<br>
[三. 重入机制的实现](#jump3)
<br>







<br><br>
## <span id="jump1">一. ReentrantLock中自定义AQS同步器的继承关系</span>

[![gf2aSe.png](https://z3.ax1x.com/2021/05/18/gf2aSe.png)](https://imgtu.com/i/gf2aSe)

ReentrantLock中通过自定义实现AQS同步器中的方法,实现了公平与非公平锁两种模式.<br>



<br><br>
## <span id="jump2">二. 公平与非公平锁的实现原理</span>

源码如下:
```
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    final void lock() {
        acquire(1);
    }
    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

再看下nonfairTryAcquire源码
```
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

明显可见,NonfairSync中的lock及tryAcquire方法实现中,都可能会出现抢占的情况,因此它非公平.<br>

而FairSync中会判断是否有更早的节点已经在排队了,如果有的话,会将当前节点按序排队,不会去抢占.<br>



<br><br>
## <span id="jump3">三. 重入机制的实现</span>

ReentrantLock采用的是独占模式,在加锁成功后会将state值+1,并设置exclusiveOwnerThread为当前获取资源成功的线程.重入加锁时会判断请求线程是否为已获取锁的线程(即是否为exclusiveOwnerThread),是则state值+1,然后获取锁成功.解锁时,只有state=0时才完成解锁,同时exclusiveOwnerThread置为null,以此实现重入机制.<br>

