---
layout:     post
title:      "something about kafka"
date:       2021-04-13 21:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---

<br>
**<font size="4">Rebalance</font>** <br>

Rebalance时,会SWT,Rebalance的Group组不消费消息


<br>
**<font size="4">位移信息</font>** <br>

老版本位移信息保存在ZK中,但是ZK并不适合频繁更新(基于主节点的ZAB协议,一种两阶段提交操作,写性能巨差),新版本中将位移信息写入kafka的内部主题'\_\_consumer_offsets'中.<br>

kafka位移提交分手动提交位移和自动提交位移.手动提交,顾名思义,就是你得控制每次去提交.自动提交就是每隔一段时间自动提交位移,当然自动提交下,每次调用poll方法的逻辑是,先提交上一批消息的位移,再拉取下一批消息.位移的提交实际上是在分区粒度上进行的,即 Consumer 需要为分配给它的每个分区提交各自的位移数据.<br>


<br>
**<font size="4">kafka的事务消息</font>** <br>

kafka中定义的事务消息,指的是一批消息中,发送给不同队列的各条消息可以保证要么全写入成功,要么全不成功,这与RocketMQ关于事务消息的定义视角不同(RocketMQ更偏重于发送消息与本地业务逻辑执行的一致性,比如发消息与写db).kafka发送事务消息需要producer端在编码上做相应调整,consumer中需要设置isolation_level参数为'read_committed',原本默认是'read_uncommitted'.因为kafka中即使消息发送写入失败,也会保存在日志中,consumer如果不设置该参数,还是能看到这些消息,造成不一致性.producer端编码调整为如下:

```

producer.initTransactions();
try {
            producer.beginTransaction();
            producer.send(record1);
            producer.send(record2);
            producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```


<br>
**<font size="4">消息日志文件</font>** <br>

kafka是每个topic_partition一个文件,但topic或者partition过多,每个文件的顺序IO,表现到磁盘上,还是随机IO.RocketMQ对这点进行了优化,是所有topic一个文件,即commitLog


<br>
**<font size="4">副本机制</font>** <br>

kafka的副本机制是基于领导者的副本机制(Leader-based),但是非leader的副本,只能用来做HA,并不能做读写分离以提高读性能.<br>

另外Follower Replica是异步的复制Leader Replica的消息的.而RocketMQ是能够配置为同步复制的.<br>

因为是Follower Replica异步复制,故kafka定义了一套自己的In-sync Replicas（ISR)机制,用以表明什么标准下,符合kafka中定义的'达到同步'标准.这个标准就是 Broker 端参数 replica.lag.time.max.ms 参数值.这个参数的含义是 Follower 副本能够落后 Leader 副本的最长时间间隔,只要一个 Follower 副本落后 Leader 副本的时间不连续超过该时间值,那么 Kafka 就认为该 Follower 副本与 Leader 是同步的,kafka将包括leader自身在内和符合同步标准的follower,用ISR集合记录,该集合是动态的,也就是其中的候选集可能一会不满足了,就被踢出,一会又满足了,再进入.<br>

而kafka要保证不丢消息,需要设置Producer端参数'acks=all',设置此参数后,发送的消息需要确保follower的异步复制消息,都复制过该消息后,才算消息提交成功.因此kafka要想做到不丢消息,性能会极其低下,就是因为上面分析的这套副本异步复制机制.<br>

