---
layout:     post
title:      "LinkedHashMap"
date:       2021-05-12 13:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---







## 导航
[一. 继承关系](#jump1)
<br>
[二. 数据结构](#jump2)
<br>
[三. 维护链表的操作](#jump3)
<br>










<br><br>
## <span id="jump1">一. 继承关系</span>

[![gdzOq1.png](https://z3.ax1x.com/2021/05/12/gdzOq1.png)](https://imgtu.com/i/gdzOq1)



<br><br>
## <span id="jump2">二. 数据结构</span>

在HashMap的基础上维护了一个双向链表.
[![gwSFsA.png](https://z3.ax1x.com/2021/05/12/gwSFsA.png)](https://imgtu.com/i/gwSFsA)



<br><br>
## <span id="jump3">三. 维护链表的操作</span>

维护链表主要使用三个方法 : afterNodeRemoval,afterNodeInsertion,afterNodeAccess.这三个方法的主要作用是,在删除,插入,获取节点之后,对链表进行维护.<br>

afterNodeRemoval:
```
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

afterNodeAccess:
```
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    //如果accessOrder为false,什么都不做,accessOrder默认就是false
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

<font color="red">所以,这里可以通过在构造LinkedHashMap的时候传入accessOrder参数为true,使其按访问顺序维护链表,即每次调用get方法访问到的节点会被维护到链表头部.</font> <br>

afterNodeInsertion:
```
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

<font color="red">LinkedHashMap中对removeEldestEntry方法的默认实现为返回false,因此不会进行节点的remove.而对插入的节点在链表中的维护则是在put时,创建新节点的方法中实现的.LinkedHashMap重写了父类的newNode和newTreeNode两个方法,在它们内部实现中加入了链表维护的逻辑</font> <br>

<font color="red">由上面对afterNodeRemoval和afterNodeInsertion的分析可知,可通过设置accessOrder属性以及重写removeEldestEntry方法,来通过LinkedHashMap实现LRU链表</font> <br>

