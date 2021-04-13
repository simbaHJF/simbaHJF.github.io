---
layout:     post
title:      "something about kafka"
date:       2021-04-13 21:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---



* Rebalance时,会SWT,Rebalance的Group组不消费消息
* 老版本位移信息保存在ZK中,但是ZK并不适合频繁更新(基于主节点的ZAB协议,一种两阶段提交操作,写性能巨差),新版本中将位移信息写入kafka的内部主题'\_\_consumer_offsets'中
* kafka中定义的事务消息,指的是一批消息中,发送给不同队列的各条消息可以保证要么全写入成功,要么全不成功,这与RocketMQ关于事务消息的定义视角不同(RocketMQ更偏重于发送消息与本地业务逻辑执行的一致性,比如发消息与写db).kafka发送事务消息需要producer端在编码上做相应调整,consumer中需要设置isolation_level参数为'read_committed',原本默认是'read_uncommitted'.因为kafka中即使消息发送写入失败,也会保存在日志中,consumer如果不设置该参数,还是能看到这些消息,造成不一致性.producer端编码调整为如下:

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
* kafka是每个topic_partition一个文件,但topic或者partition过多,每个文件的顺序IO,表现到磁盘上,还是随机IO.RocketMQ对这点进行了优化,是所有topic一个文件,即commitLog