---
layout:     post
title:      "RocketMQ Broker是如何持久化存储消息的"
date:       2021-04-08 11:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. CommitLog](#jump1)
<br>
[二. MessageQueue在数据存储中是体现在哪里](#jump2)
<br>
[三. 消息写入与刷盘策略](#jump3)
<br>











<br><br>
## <span id="jump1">一. CommitLog</span>

当Broker接收到生产者发送来的消息时,首先第一步,它会把这个消息直接写入磁盘上的一个日志文件,叫做<font color="red">CommitLog</font>.直接顺序写入,如下图:
[![cJYj3D.png](https://z3.ax1x.com/2021/04/08/cJYj3D.png)](https://imgtu.com/i/cJYj3D)

CommitLog是很多磁盘文件,每个文件限定最多1G,Broker收到消息之后就直接追加写入这个文件的末尾,如果一个CommitLog写满了1G,就会创建一个新的CommitLog文件.RocketMQ 的所有主题的消息都存在 同一CommitLog 中,并不向kafka,以topic-partition来区分.log文件. <br>

文件名以起始偏移量命名,固定 20 位,不足则前面补0,比如 00000000000000000000 代表了第一个文件,第二个文件名就是 00000000001073741824,表明起始偏移量为 1073741824,以这样的方式命名用偏移量就能找到对应的文件.这点与kafka的.log文件命名方式是一致的
[![cJcfXt.png](https://z3.ax1x.com/2021/04/08/cJcfXt.png)](https://imgtu.com/i/cJcfXt)


<br><br>
## <span id="jump2">二. MessageQueue在数据存储中是体现在哪里</span>

在Broker中,对Topic下的每个MessageQueue都会有一系列的ConsumeQueue文件.这是什么意思呢?就是在Broker的磁盘上,会有下面这种格式的一系列文件:
```
$HOME/store/consumequeue/{topic}/{queueId}/{fileName}
```

每个Topic在这台Broker上都会有一些MessageQueue,上面路径中会看到,{topic}指代的就是某个topic,{queueId}指代的就是某个MessageQueue
[![cJcR1A.png](https://z3.ax1x.com/2021/04/08/cJcR1A.png)](https://imgtu.com/i/cJcR1A)

<font color="red">这里针对上图有一点说明: 在本地测试时,只有一个broker,而RocketMQ默认为每个topic分配4个队列,所以在左下图中看到了4个文件夹.事实上,生产环境中,可能一个broker下的一个topic下只有一个队列文件夹,但文件夹下可能有多个队列文件,因为写满会新建</font> <br>

当Broker收到一条消息,将其写入了CommitLog之后,其实它同时会将这条消息在CommitLog中的物理位置,也就是一个文件偏移量offset,写入到这条消息所属的MessageQueue对应的ConsumeQueue文件中去.<br>

<font color="red">所以实际上,ConsumeQueue中存储的是一个一个消息在CommitLog文件中的物理位置,也就是offset.当然,实际上ConsumeQueue中存储的每条数据不只是消息在CommitLog中的offset偏移量,还包含了消息的长度,以及tag hashcode,一条数据是20个字节,每个ConsumeQueue文件保存30万调数据,大约每个文件是5.72M,写满后会新建一个新的ConsumeQueue文件.</font> <br>

所以,Topic的每个MessageQueue都对应了Broker机器上的多个ConsumeQueue文件,保存了这个MessageQueue的所有消息在CommitLog文件中的物理位置,也就是offset偏移量.<br>



<br><br>
## <span id="jump3">三. 消息写入与刷盘策略</span>

顺序写 + pageCache <br>

顺序写来提升写入性能,避免磁盘的随机访问,磁盘顺序写的性能在一定程度上是可以趋近于内存的,尤其是针对SSD硬盘.<br>

刷盘策略方面,存在两种机制:
* 异步刷盘: 消息写入CommitLog时,只是写到了OS的pageCache中还未刷盘,就直接返回给producer端ack,真正的刷盘依赖OS.
	* 优点: 高性能
	* 缺点: 如果Broker宕机,还没来得及进行刷盘的消息,会丢失
* 同步刷盘: 消息写入每次必须刷盘到物理磁盘中完成,才会返回给Producer端ack
	* 优点: 即便Broker宕机,也不会丢消息
	* 缺点: 性能急剧降低
