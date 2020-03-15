---
layout:     post
title:      "从WeakReference说起,聊聊ThreadLocal和WeakHashMap的内存泄漏问题"
date:       2019-06-09 16:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---





Java中存在4中引用类型,它们由强到弱依次是：强引用、软引用、弱引用、虚引用.
*  强引用----通常我们通过new来创建一个新对象时返回的引用就是一个强引用,若一个对象通过一系列强引用可到达,它就是强可达的(strongly reachable),那么它就不被回收.
*  软引用,弱引用----这里先简单提一下二者的区别,后面详细聊一下弱引用.软引用和弱引用的区别在于,若一个对象是弱引用可达,无论当前内存是否充足它都会被回收,而软引用可达的对象在内存不充足时才会被回收,因此软引用要比弱引用“强”一些.
*  虚引用----虚引用是Java中最弱的引用,那么它弱到什么程度呢?它是如此脆弱以至于我们通过虚引用甚至无法获取到被引用的对象,虚引用存在的唯一作用就是当它指向的对象被回收后,虚引用本身会被加入到引用队列中,用作记录它指向的对象已被销毁.

下面就来具体聊一下弱引用
<br>
## 弱引用
首先,弱引用这是一个和垃圾回收有关的概念.

下面一段话翻译自WIKIPEDIA:
在计算机程序中,一个弱引用是这样一种引用:它不能保护被引用的对象不被垃圾收集器回收,这点与强引用不同(强引用的对象是不会被垃圾收集器回收的).一个只被弱引用所引用的对象(任意一条可以达到该对象的引用链都至少包括一条弱引用连接)是被认为弱可达的,这个对象可以被当做不可达一样处理,所以它可能在任何时候被(垃圾收集器)回收.

 Java中的弱引用,就是java.lang.ref.WeakReference<T>类,首先来看下JDK对该类所做的说明:
弱引用对象的存在不会阻止它所指向的对象被垃圾回收器回收.弱引用最常见的用途是实现规范映射(canonicalizing mappings).
假设垃圾收集器在某个时间点确定一个对象是弱可达的(weakly reachable,可以简单理解为当前指向它的全都是弱引用),这时垃圾收集器会清除所有指向该对象的弱引用,然后垃圾收集器会把这个弱可达对象标记为可终结的(finalizable),这样他们随后就会被回收.与此同时或稍后,垃圾收集器会把那些刚清除的弱引用放入创建弱引用对象时所登记到的引用队列中.
<br>
<br>

#####  1.  弱引用的使用场景举例(摘自WIKIPEDIA)
```
import java.lang.ref.WeakReference;

public class ReferenceTest {
    public static void main(String[] args) throws InterruptedException {
            //一个弱引用
            WeakReference<String> r = new WeakReference("I'm here");

            //一个强引用
            String sr = new String("I'm here");
            System.out.println("before gc: r=" + r.get() + ", sr=" + sr);
            System.gc();
            Thread.sleep(100);

            // only r.get() becomes null
            System.out.println("after gc: r=" + r.get() + ", sr=" + sr);

    }
}
```
如果一个弱引用被创建,然后在程序的其他地方通过get()方法来获取被弱引用关联的对象,弱引用的强度不足以阻止垃圾回收,所以它或许(前提是,没有强引用指向该对象的话)在调用get()方法时,突然开始返回null值.
弱引用的另一个应用是写缓存.例如,使用一个弱hashmap做缓存时,一个弱hashmap能够在缓存中通过弱引用存储大量的被引用对象.当垃圾收集器运行的时候----比如当应用程序的内存使用量相当高的时候----那些被缓存的对象(这些对象不再被其他对象直接引用,也就是不存在强引用指向它们)将会被从缓存中清除.
<br>
<br>


#####  2.  弱引用的原理
先来看一下jkd中对弱引用的实现:
```
public class WeakReference<T> extends Reference<T> {
    /**
     * Creates a new weak reference that refers to the given object.  The new
     * reference is not registered with any queue.
     *
     * @param referent object the new weak reference will refer to
     */
    public WeakReference(T referent) {
        super(referent);
    }
    /**
     * Creates a new weak reference that refers to the given object and is
     * registered with the given queue.
     *
     * @param referent object the new weak reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     */
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```
为了理解弱引用原理,我们需要查看三个类WeakReference,Reference,ReferenceQueue.可以看到WeakReference继承了Reference,整个WeakReference的代码十分简单,主要的逻辑由Reference实现,WeakReference的两个构造器也是调用了其父类Reference的构造器.
下面来看下Reference的两个构造方法实现:
```
public abstract class Reference<T> {
    Reference(T referent) {
        this(referent, null);
    }
    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}
```
其构造器中有两个十分重要的属性,第一个参数是一个引用对象.在Reference中,回收器线程将对这个对象进行监视,从而决定是否将Reference放入ReferenceQueue中.而这个ReferenceQueue正是传递给构造器的第二个属性.Reference对象为了自己专门定义了四个内部状态:Active-Pending-Enqueued-Inactive.
*  Active:  对象的一种状态表明其将被垃圾收集器特殊对待.当某个时刻,垃圾收集器发现对象引用的可达性状态变达到了某种条件时,它会改变对象的状态为Pending或者Inactive,这取决于当对象创建时,对象是否被注册到一个queue中(解释一下就是对象创建时是否传入了第二个参数ReferenceQueue).在前一种情况下,它还将实例添加到pending-Reference队列中(这个pending队列是所有Reference对象共享的,创建对象时注册不同ReferenceQueue的对象,都会使用这一个pending队列).新创建的对象是Active状态.
*  Pending:  pending-Reference队列中的元素等待着被Reference-handler(引用处理器)线程处理入队(这个要入的队列就是构造器第二个参数传入的ReferenceQueue).未注册的对象永远不会处于这种状态.
*  Enqueued:  处于ReferenceQueue中的元素的状态,这些元素在创建的时候,进行了注册(也就是传入了构造器第二个参数ReferenceQueue).当一个对象从它注册的ReferenceQueue中被移除时,他就是Inactive状态.未注册对象永远不会处于这种状态.
*  Inactive:  没什么可以做了.一旦一个对象变成了Inactive状态,它的状态就再也不会改变了.

当引用对象被创建那一刻,Reference对象的状态为Active.回收器开始运作,当可达性发生变化达到某种条件时,Reference对象状态将被转化为Pending(这部分是由JVM实现的).同时,Reference对象将被加入一个Pending队列(源码中的实现为链表).Reference类还启动一个守护线程ReferenceHandle.这个线程负责将Pending队列中的Reference对象加入ReferenceQueue队列中.可以说ReferenceQueue中的Reference对象已经是无用对象.ReferenceQueue可以看作是一个Reference链表.同时加入其中的Reference对象已经不可达,垃圾回收器将对其进行自动回收.

这里再提一下ReferenceHandle,该引用处理器是所有引用对象所共享的.看下面的代码:
```
public abstract class Reference<T> {
    //..................
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
    }
    //..................
}
```
在Reference类初始化时会调用一次该静态块,在此静态块中启动了一个守护线程ReferenceHandler,这个守护线程会监视pending队列(上文提到过这个队列,也是各引用对象共享的),它会将pending队列中的引用对象入队到引用对象创建时注册的ReferenceQueue中.

说了这么多,Java的弱引用应该解释的差不多了,下面来说ThreadLocal和WeakHashMap

##  ThreadLocal
ThreadLocal中有一个内部类ThreadLocalMap,在此内部类中实现了一个重要的数据结构ThreadLocalMap.Entry,以及一个Entry[],如下:
```
static class ThreadLocalMap {
    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    //...............
    //...............
    private Entry[] table;
}
```
Entry结构就是一种K/V的结构,它的Key是ThreadLocal对象,Value是线程中为该ThreadLocal对象关联的值对象.可以看到,Entry继承了WeakReference,也就是说,它是一个弱引用,这里需要注意的是,它的弱引用是实现在它的Key上的,也就是ThreadLocal对象,它的Value并不是弱引用对象.所以它的弱引用,体现在其Entry结构的Key上,而不是整个Entry结构,也不是Entry下的Value结构,这点要特别注意.

ThreadLocalMap类中,还有个一Entry的列表属性table,这个属性是用作什么的呢?下面来解释一下:
每个Thread对象内部都有一个ThreadLocal.ThreadLocalMap类型的属性,这是因为一个线程可能会用到多个ThreadLocal,所以就将这些所有用到的ThreadLocal对象都交由ThreadLocal.ThreadLocalMap下的Entry[]来管理.在调用ThreadLocal对象的set方法存入当先线程副本时,就需要知道放到哪个位置的Entry下,而这个算index的逻辑也是通过哈希值和table长度来共同决定的,这时就会有发生哈希碰撞的情况,那么ThreadLocalMap是如何解决哈希碰撞的呢,它并没有采用我们熟知的HashMap那种数组加链表或红黑树的这种链地址法来解决哈希碰撞,而是采用开放地址法.
```
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
介绍完ThreadLocal的基本情况,来看看为什么ThreadLocal会发生内存泄漏的问题.考虑这样一种情况,当创建很多ThreadLocal对象,或者有大量线程操作一个ThreadLocal对象时,当然这两种情况要加一个前提,那就是线程未死亡.当对ThreadLocal对象的强引用被置null后,由于Entry对象的Key也就是ThreadLocal实现了弱引用,GC会对其进行回收,但是Entry结构中的Value对象却是一个强引用,其被Entry的v属性所引用,从而造成无法被回收,但是由于Key已经被回收了,Value也就找不到了(但它还没法被回收,切切实实的还在内存堆中),这时就造成了内存泄漏.
##  WeakHashMap
WeakHashMap内部也实现了一个自己的Entry结构:
```
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;
    /**
     * Creates new entry.
     */
    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
    //.........
}
```
可以看到,和ThreadLocal中的Entry结构很类似,也是对Entry的Key做了弱引用实现,但有一点不同的是,它生成弱引用对象的时候进行了注册,传入了ReferenceQueue.

下面来聊聊WeakHashMap的内存泄漏问题:WeakHashMap很少引发内存泄漏,除非使用错误,WeakHashMap中的key被value直接或间接所引用.在使用正确的情况下,WeakHashMap中的数据在key没有被强引用的情况下,回收器可以正确回收整个Entry的内存.这是为什么呢?WeakHashMap专门实现了下面这个方法,它会对创建弱引用Key时注册的ReferenceQueue进行翻找,把其对应的整个Entry都回收.
```
/**
 * Expunges stale entries from the table.
 */
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);
            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

