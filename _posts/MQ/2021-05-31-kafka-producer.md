---
layout:     post
title:      "kafka producer"
date:       2021-05-31 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---





## 导航
[一. kafka producer整体架构](#jump1)
<br>
[二. 流程说明](#jump2)
<br>
[三. 重要的生产者参数](#jump3)
<br>











<br><br>
## <span id="jump1">一. kafka producer整体架构</span>

[![2em4fg.png](https://z3.ax1x.com/2021/05/31/2em4fg.png)](https://imgtu.com/i/2em4fg)



<br><br>
## <span id="jump2">二. 流程说明</span>

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和 Sender线程 (发送线 程)。在主线程中由 KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作 用之后缓存到消息累加器( RecordAccumulator，也称为消息收 集器〉中。 Sender 线程负责从 RecordAccumulator中获取消息并将其发送到 Kafka中。 <br>

RecordAccumulator 主要用来缓存消息 以便 Sender 线程可以批量发送，进而减少网络传输 的资源消耗以提升性能 。<br>

ProducerBatch 中可以包含一至多个 ProducerRecord。<br>

如果消息 ProducerRecord 中指定了 partition 字段， 那么就不需要分区器的作用 ，因 为 partition 代表的就是所要发往的分区号。如 果 消 息 ProducerRecord 中没有指定 partition 字段，那么就需要依赖分区器 ， 根据 key这个字段来计算 partition 的值。分区器的作用就是为消息分配分区。<br>

消息发送方式分为三种:
* one way
* 同步
* 异步



<br><br>
## <span id="jump3">三. 重要的生产者参数</span>

<br>
**<font size="5">acks</font>** <br>

这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消 息是成功写入的。 acks 是生产者客户端中一个非常重要 的参数 ，它涉及消息的可靠性和吞吐 量之间的权衡.acks参数有3种类型的值:
*  acks = 1。默认值即为 1
    >   生产者发送消息之后，只要分区的 leader副本成功写入消 息，那么它就会收到来自服务端的成功响应 。 如果消息无法写入 leader 副本，比如在 leader 副本崩溃、重新选举新的 leader 副本的过程中，那么生产者就会收到一个错误 的响应，为了避免消息丢失，生产者可以选择重发消息 。如果消息写入 leader 副本并 返回成功响应给生产者，且在被其他 follower 副本拉取之前 leader 副本崩溃，那么此 时消息还是会丢失，因为新选举的 leader 副本中并没有这条对应的消息 。 acks 设置为 l，是消息可 靠性和吞吐量之 间的折中方案。
*  acks = 0,也即one way方式
    >   生产者发送消 息之后不需要等待任何服务端的响应 。如果在消息从发送到 写入 Kafka 的过程中出现某些异常，导致 Kafka 并没有收到这条消息，那么生产者也 无从得知，消息也就丢失了。在其他配置环境相同的情况下， acks 设置为 0 可以达 到最大的吞吐量。<br>
*  acks =一l 或 acks =all.  能保证高可用且leader副本宕机安全.
    >   生产者在消息发送之后，需要等待 ISR 中的所有副本都成功 写入消息之后才能够收到来自服务端的成功响应。在其他配置环境相同的情况下， acks 设置为 1 (all)可以达到最强的可靠性。但这并不意味着消息就一定可靠，因 为 ISR 中可 能只有 leader 副本，这样就退化成了 acks=1 的情况。要获得更高的消息 可靠性需要配合 min.insync.replicas 等参数的联动


<br><br>
**<font size="5">retries 和 retry.backoff.ms</font>** <br>

retries 参数用来配置生产者重试的次数，默认值为 0，即在发生异常的时候不进行任何 重试动作。消息在从生产者发出到成功写入服务器之前可能发生一些临时性的异常， 比如网 络 抖动、 leader副本的选举等，这种异常往往是可以自行恢复的，生产者可以通过配置 retries 大于 0 的值，以此通过 内部重试来恢复而不是一昧地将异常抛给生产者的应用程序。 如果重试 达到设定的 次数 ，那么生产者就会放弃重试并返回异常.<br>

重试还和另一个参数 retry.backoff.ms 有关，这个参数的默认值为 100， 它用来设定 两次重试之间的时间间隔，避免无效的频繁重试.<br>


<br>
**<font size="5">compression.type</font>** <br>

这个参数用来指定消息的压缩方式，默认值为“none”，即默认情况下，消息不会被压缩。该参数还可以配置为“gzip”, “snappy” 和“lz4”。 对消息进行压缩可以极大地减少网络传输 量、降低网络 I/0，从而提高整体的性能。消息压缩是一种使用时间换空间的优化方式，如果对 时延有一定的要求，则不推荐对消息进行压缩。<br>


<br>
**<font size="5">linger.ms</font>** <br>

这个参数用来指定生产者发送 ProducerBatch 之前等待更多消息( ProducerRecord)加入 ProducerBatch 的时间，默认值为 0。生产者客户端会在 ProducerBatch 被填满或等待时间超过 linger .ms 值时发迭出去。增大这个参数的值会增加消息的延迟，但是同时能提升一定的吞 吐量。 这个linger.ms参数与TCP协议中的Nagle算法有异曲同工之妙。<br>
