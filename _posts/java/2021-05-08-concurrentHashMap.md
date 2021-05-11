---
layout:     post
title:      "ConcurrentHashMap"
date:       2021-05-08 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---






## 导航
[一. JDK7](#jump1)
<br>
[二. JDK8](#jump2)
<br>









<br><br>
## <span id="jump1">一. JDK7</span>

ConcurrentHashMap使用分段锁技术,将整个数据结构分段(默认为16段,可以在创建时传入构造方法以更改其值)进行存储,然后给每一段数据配一把锁(继承ReentrantLock),当一个线程占用锁访问其中一个段的数据的时候,其他段的数据仍然能被其他线程访问,能够实现真正的并发访问.一旦segment分段数量初始化确定之后,在ConcurrentHashMap的生命周期中,其值就不会再变更了,即便发生resize,扩容的也只是segment下table的容量.下图为JDK7的数据结构
[![gGDa8S.png](https://z3.ax1x.com/2021/05/08/gGDa8S.png)](https://imgtu.com/i/gGDa8S)


<br>
**<font size="5">put过程</font>** <br>

1. 首先对key进行第1次hash,通过hash值确定segment的位置
2. 然后在segment内进行操作,获取锁
3. 获取当前segment的HashEntry数组后对key进行第2次hash,通过hash值确定在HashEntry数组的索引位置
4. 通过继承ReentrantLock的tryLock方法尝试去获取锁,如果获取成功就直接插入相应的位置,如果已经有线程获取该Segment的锁,那当前线程会以自旋的方式去继续的调用tryLock方法去获取锁,超过指定次数就挂起,等待唤醒
5. 然后对当前索引的HashEntry链进行遍历,如果有重复的key,则替换;如果没有重复的,则插入到链头
6. 释放锁

> get操作和put操作类似,也是要两次hash.但是get操作的concurrenthashmap不需要加锁,原因是将存储元素都标记了volatile



<br>
**<font size="5">size过程</font>** <br>

1. size操作就是遍历了两次所有的Segments,每次记录Segment的modCount值,然后将两次的modCount进行比较,如果相同,则表示期间没有发生过写入操作,就将原先遍历的结果返回
2. 如果经判断发现两次统计出的modCount并不一致,要重新启用全部segment加锁的方式来进行count的获取和统计了,这样在此期间每个segement都被锁住,无法进行其他操作(增删改,不影响读),统计出的count自然很准确

> 在写操作put,remove,扩容的时候,会对Segment加锁,只影响当前Segment,其他的Segment还是可以并发的



<br><br>
## <span id="jump2">二. JDK8</span>

JDK8的ConcurrentHashMap的数据结构已经接近对应版本的HashMap,了解Hashmap的结构,就基本了解了Concurrenthashmap了,只是增加了同步的操作来控制并发.从JDK7版本的ReentrantLock+Segment+HashEntry,到JDK8版本中synchronized+CAS+HashEntry+红黑树.底层依然是"数组+链表+红黑树"的方式,但是为了做到并发,又增加了很多辅助的类

[![gGoC8J.png](https://z3.ax1x.com/2021/05/08/gGoC8J.png)](https://imgtu.com/i/gGoC8J)


<br>
**<font size="5">重要属性</font>** <br>


<br>
**<font size="4">sizeCtl</font>** <br>

它是一个控制标识符,在不同的地方有不同用途,而且它的取值不同,也代表不同的含义
* 负数代表正在进行初始化或扩容操作
	* -1代表正在初始化
	* -N 表示有N-1个线程正在进行扩容操作
* 正数或0
	* table为null时,值为默认0或者创建table时,其应该的初始大小
	* table初始化之后,其值标识下一次进行扩容的大小.这一点类似于扩容阈值的概念.它的值始终是当前ConcurrentHashMap容量的0.75倍


<br>
**<font size="4">Node</font>** <br>

Node是最核心的内部类,它包装了key-value键值对,所有插入ConcurrentHashMap的数据都包装在这里面.它与HashMap中的定义很相似,但是有一些差别
* 它对value和next属性设置了volatile同步锁
* 它不允许调用setValue方法直接改变Node的value域
* 它增加了find方法辅助map.get()方法

Node节点,是所有节点的父类,可以单独放入桶内,也可以作为链表的头放入桶内


<br>
**<font size="4">TreeNode</font>** <br>

树节点类,另外一个核心的数据结构.当链表长度过长的时候,会转换为TreeNode.但是与HashMap不相同的是,它并不是直接转换为红黑树,而是把这些结点包装成TreeNode放在TreeBin对象中,由TreeBin完成对红黑树的包装.TreeNode在ConcurrentHashMap继承自ConcurrentHashMap.Node类<br>

TreeNode节点,继承自Node,是红黑树的节点,此节点不能直接放入桶内,只能是作为红黑树的节点,由TreeBin链接,TreeBin会指向红黑树的根结点


<br>
**<font size="4">TreeBin</font>** <br>

这个类并不负责包装用户的key、value信息,而是包装的很多TreeNode节点.它代替了TreeNode的根节点,也就是说在实际的ConcurrentHashMap"数组"中,存放的是TreeBin对象,而不是TreeNode对象,这是与HashMap的区别.另外这个类还带有了读写锁<br>

TreeBin节点,TreeNode的代理节点,可以放入桶内,这个节点下面可以连接红黑树的根节点,所以叫做TreeNode的代理节点.该结点提供了一系列红黑树相关的操作,以及加锁、解锁操作


<br>
**<font size="4">ForwardingNode</font>** <br>

一个用于连接两个table的节点类.它包含一个nextTable指针,用于指向下一张表.而且这个节点的key value next指针全部为null,它的hash值为-1. 这里面定义的find的方法是从nextTable里进行查询节点,而不是以自身为头节点进行查找.<br>

A node inserted at head of bins during transfer operations.

1. ForwardingNode是一种临时结点,在扩容进行中才会出现,hash值固定为-1,且不存储实际数据.
2. 如果旧table数组的一个hash桶中全部的结点都迁移到了新table中,则在这个桶中放置一个ForwardingNode.
3. 读操作碰到ForwardingNode时,将操作转发到扩容后的新table数组上去执行;写操作碰见它时,则尝试帮助扩容,扩容是支持多线程一起扩容的.
4. 提供了在新的数组nextTable上进行查找的方法find


<br>
**<font size="4">ReservationNode</font>** <br>

1. 保留结点
2. hash值固定为-3,不保存实际数据
3. 只在computeIfAbsent和compute这两个函数式API中充当占位符加锁使用


<br>
**<font size="5">size方法</font>** <br>

两个重要变量:
* baseCount用于记录节点的个数,是个volatile变量
* counterCells是一个辅助baseCount计数的数组,每个counterCell存着部分的节点数量,这样做的目的就是尽可能地减少冲突,counterCell类使用了 @sun.misc.Contended 标记,内部一个 volatile变量,以此避免"伪共享"问题

ConcurrentHashMap节点的数量 = baseCount+counterCells每个cell记录下来的节点数量.由于JDK8在统计这个数量的时候并没有进行加锁,所以这个结果并不是绝对准确的<br>

更新size的过程: 总体的原则就是,先尝试更新baseCount,失败再利用CounterCell
1. 通过CAS尝试更新baseCount,如果更新成功则完成,如果CAS更新失败会进入下一步
2. 线程通过随机数ThreadLocalRandom.getProbe() & (n-1) 计算出在counterCells数组的位置,如果不为null,则CAS尝试在couterCell上直接增加数量,如果失败,counterCells数组会进行扩容为原来的两倍,继续随机,继续添加


<br>
**<font size="5">put方法</font>** <br>

1. 如果没有初始化,先初始化table
2. table[i]对应的桶为空(没有hash冲突),直接占用table[i],CAS插入
3. ForwardingNode结点,说明此时table正在扩容,则尝试协助进行数据迁移
4. 如果存在hash冲突,就加synchronized锁来保证线程安全,这里有两种情况
	* 当table[i]的结点类型为Node——链表结点时,就会将新结点以"尾插法"的形式插入链表的尾部
	* 当table[i]的结点类型为TreeBin——红黑树代理结点时,就会将新结点通过红黑树的插入方式插入
5. 插入完后如果是链表且该链表的数量大于阈值8,就要先转换成黑红树的结构
6. 如果添加成功就调用addCount方法统计size,并且检查是否需要扩容


<br>
**<font size="5">get方法</font>** <br>

1. 计算hash值,定位到该table的bucket索引位置,如果首节点(即bucket位)和要get的entry匹配,就返回
2. 如果遇到扩容的时候(桶节点为ForwardingNode),会调用标志正在扩容节点ForwardingNode的find方法,查找该节点,匹配就返回
3. 以上都不符合的话,即既没有在扩容,又不等于首节点(bucket位),则按链表遍历,找到匹配的就返回,否则最后就返回null


<br>
**<font size="5">扩容</font>** <br>

扩容时机:
1. 使用put()添加元素时会调用addCount(),内部检查sizeCtl看是否需要扩容
2. tryPresize()被调用,此方法被调用有两个调用点
	* 链表转红黑树(put()时检查)时如果table容量小于64(MIN_TREEIFY_CAPACITY),则会触发扩容
	* 调用putAll()之类一次性加入大量元素,会触发扩容


数据迁移方法transfer:
```
/**
 * tab,    旧table
 * nextTAB 扩容后的table
 */
private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) {
    int n = tab.length, stride;
    //每个线程负责迁移table一个区间段的桶的个数，最少是16个
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //首次扩容
    if (nextTab == null) {            // initiating
        try {
            //创建新的table,默认n*2
            Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; //扩容总进度，>=transferIndex的桶都已分配出去
    }
    int nextn = nextTab.length;
    //创建扩容节点，当某个桶迁移完成后，放入table[i],标记桶扩容完成
    ForwardingNode<K, V> fwd = new ForwardingNode<K, V>(nextTab);
    boolean advance = true; //为true,表示当前桶迁移完成，可以继续处理下一个桶
    boolean finishing = false; // 最后一个数据迁移的线程将该值置为true，进行扩容的收尾工作
    //i桶索引，bound就是线程要处理的另一个区间边界
    for (int i = 0, bound = 0; ; ) {
        Node<K, V> f;
        int fh;
        //定位本轮处理区间【transferIndex-1，transferIndex-stride】
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            } else if (U.compareAndSwapInt
                    (this, TRANSFERINDEX, nextIndex,
                            nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //Case1,最后一个迁移线程或者是线程出现了冲突，导致了i<0
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) { //迁移完成
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //扩容线程数减1,当前线程任务已执行完成
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //判断是否最后一个迁移线程，不是则退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                //最后一个线程，还原i值，重新进行检查，是否全部迁移完成，应该所有桶都是ForwardingNode
                i = n; // recheck before commit
            }
        }
        // CASE2：旧桶本身为null，不用迁移，放一个ForwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
            //CASE3：该旧桶已经迁移完成，直接跳过，hash==moved 代表ForwardingNode
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // CASE4：该旧桶未迁移完成，进行数据迁移，加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K, V> ln, hn;
                    if (fh >= 0) { //桶是链表，迁移链表
                        /**
                         * 下面的过程会将旧桶中的链表分成两部分：ln链和hn链
                         * ln链会插入到新table的槽i中，hn链会插入到新table的槽i+n中
                         */
                        int runBit = fh & n;
                        Node<K, V> lastRun = f;
                        for (Node<K, V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        } else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K, V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash;
                            K pk = p.key;
                            V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K, V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K, V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);  // ln链表存入新桶的索引i位置
                        setTabAt(nextTab, i + n, hn); // hn链表存入新桶的索引i+n位置
                        setTabAt(tab, i, fwd); // 设置ForwardingNode占位
                        advance = true; // 表示当前旧桶的结点已迁移完毕
                    } else if (f instanceof TreeBin) { //红黑树迁移
                        TreeBin<K, V> t = (TreeBin<K, V>) f;
                        TreeNode<K, V> lo = null, loTail = null;
                        TreeNode<K, V> hi = null, hiTail = null;
                        /**
                         * 先以链表方式遍历，复制所有结点，然后根据高低位组装成两个链表；
                         * 然后看下是否需要进行红黑树转换，最后放到新table对应的桶中
                         */
                        int lc = 0, hc = 0;
                        for (Node<K, V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K, V> p = new TreeNode<K, V>
                                    (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            } else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 判断是否需要进行 红黑树 <-> 链表 的转换
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K, V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K, V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);// 设置ForwardingNode占位
                        advance = true; // 表示当前旧桶的结点已迁移完毕
                    }
                }
            }
        }
    }
}
```

helpTransfer方法:
```
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

添加、删除节点之处都会检测到table的第i个桶,是ForwardingNode的话会调用helpTransfer()方法协助迁移.


<br>
**<font size="5">并发扩容总结</font>** <br>

1. 单线程新建nextTable,新容量一般为原table容量的两倍
2. 每个线程想增/删元素时,如果访问的桶是ForwardingNode节点,则表明当前正处于扩容状态,协助一起扩容完成后再完成相应的数据更改操作
3. 扩容时将原table的所有桶倒序分配,每个线程每次最小分配16个桶,防止资源竞争导致的效率下降.单个桶内元素的迁移是加锁的,但桶范围处理分配可以多线程,在没有迁移完成所有桶之前每个线程需要重复获取迁移桶范围,直至所有桶迁移完成
4. 一个旧桶内的数据迁移完成但不是所有桶都迁移完成时,查询数据委托给ForwardingNode结点查询nextTable完成(find方法）
5. 迁移过程中sizeCtl用于记录参与扩容线程的数量,全部迁移完成后sizeCtl更新为新table容量的0.75倍
