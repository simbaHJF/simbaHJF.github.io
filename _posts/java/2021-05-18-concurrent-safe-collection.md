---
layout:     post
title:      "一些常用并发安全队列的实现原理"
date:       2021-05-18 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---








## 导航
[一. CopyOnWriteArrayList](#jump1)
<br>
[二. LinkedBlockingQueue](#jump2)
<br>
[三. ArrayBlockingQueue](#jump3)
<br>








<br><br>
## <span id="jump1">一. CopyOnWriteArrayList</span>

基于写时复制思想,适用于读多写少的场景.<font color="red">这点需要注意,写时复制机制一定不要用在并发写的场景下.</font> <br>

add方法源码:
```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

写入原理总结:<br>

1. 获取锁
2. 将原数组复制一份作为新数组
3. 新数组中添加新元素
4. 替换原数组
5. 释放锁

因此,写时复制思想,在写的时候一定是串行的,并行会有数据问题.<br>

读取方法如下:
```
public E get(int index) {
    return get(getArray(), index);
}


private E get(Object[] a, int index) {
    return (E) a[index];
}
```

读取时,不需要加锁,直接读取,因此CopyOnWriteArrayList属于弱一致性,在一个线程写入新元素,其他线程并发读取时,并不能立即读到,需要等待新数组替换完老数组后才能读到.<br>



<br><br>
## <span id="jump2">二. LinkedBlockingQueue</span>

offer及take方法,源码如下:
```
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

这里存在putLock及takeLock,两把锁,并不是读写锁.<br>

写入时,首先需要获取putLock锁,然后执行写入,完成后释放锁;读取时,首先需要获取takeLock,然后执行读取,完成后释放锁.<br>



<br><br>
## <span id="jump3">三. ArrayBlockingQueue</span>

源码奉上:
```
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

这里只有一把锁,因此,写入时不能读取,读取时不能写入.<br>

另外通过count来记录当前队列中的元素个数.并通过检查count来判断是否能够写入或读取<br>

