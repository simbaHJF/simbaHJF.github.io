---
layout:     post
title:      "RocketMQ rebalance"
date:       2021-04-09 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---






## 导航
[一. 什么是Rebalance](#jump1)
<br>
[二. Rebalance的触发场景](#jump2)
<br>
[三. Consumer Rebalance机制](#jump3)
<br>










<br><br>
## <span id="jump1">一. 什么是Rebalance</span>

Rebalance(再均衡)机制指的是:将一个Topic下的多个队列(或称之为分区),在同一个消费者组(consumer group)下的多个消费者实例(consumer instance)之间进行重新分配.<br>



<br><br>
## <span id="jump2">二. Rebalance的触发场景</span>

* 订阅Topic的队列数量变化
* 消费者组信息变化

本文中,暂且将队列信息和消费者组信息称之为``Rebalance元数据``,Broker负责维护这些元数据,并在二者信息发生变化时,以某种通知机制告诉消费者组下的所有实例,需要进行Rebalance.从这个角度说,Broker在Rebalance过程中,是一个协调者或者说通知者的角色,真正执行Rebalance的过程,是在Consumer客户端完成的.<br>

Broker之所以能维护Rebalance元数据,以及通知消费组下的所有实例,是因为每个消费实例与每个其消费Topic下对应的Broker都建立有TCP长连接.<br>



<br><br>
## <span id="jump3">三. Consumer Rebalance机制</span>

Broker是通知每个消费者各自Rebalance,即每个消费者自己给自己重新分配队列,而不是Broker将分配好的结果告知Consumer.<br>

从这个角度,RocketMQ与Kafka Rebalance机制类似,二者Rebalance分配都是在客户端进行,不同的是:
* 会在消费者组的多个消费者实例中,选出一个作为Group Leader,由这个Group Leader来进行分区分配,分配结果通过Cordinator(特殊角色的broker)同步给其他消费者.相当于Kafka的分区分配只有一个大脑,就是Group Leader
* 每个消费者,自己负责给自己分配队列,相当于每个消费者都是一个大脑

这里针对RocketMQ的Rebalance机制,先提出两个关键问题,随着对Rebalance机制的分析,会回答这两个问题
1. 每个消费者自己给自己分配,如何避免脑裂问题?
	> 也即因为每个消费者都不知道其他消费者分配的结果,如何确保一个队列不会分给多个消费者呢(因为RocketMQ中的机制,一个MessageQueue规定只能分配给一个消费者,但一个消费者可以被分配多个MessageQueue)
2. 如果某个消费者没有收到Rebalance通知怎么办?
	> 这里第二个问题直接回答: 每个消费者都会定时触发Rebalance,以避免Rebalance通知丢失


<br>
**<font size="4">Rebalance的触发时机</font>** <br>

* 在启动时,消费者会立即向所有Broker发送一次发送心跳(HEART_BEAT)请求,Broker则会将消费者添加由ConsumerManager维护的某个消费者组中.然后这个Consumer自己会立即触发一次Rebalance
* 在运行时,消费者接收到Broker通知会立即触发Rebalance,同时为了避免通知丢失,会周期性触发Rebalance
* 当停止时,消费者向所有Broker发送取消注册客户端(UNREGISTER_CLIENT)命令,Broker将消费者从ConsumerManager中移除,并通知其他Consumer进行Rebalance


<br>
**<font size="4">RocketMQ Rebalance特点</font>** <br>

只要一个消费者组需要Rebalance,那么这台机器上启动的所有其他消费者,也都要进行Rebalance
```
public void doRebalance() {
    //迭代每个consumer，进行rebalance
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                //逐一触发Rebalance
                impl.doRebalance();
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
```

RocketMQ是按照Topic维度进行Rebalance的
```
public void doRebalance(final boolean isOrder) {
    //1 迭代当前consumer订阅的每一个topic，逐一进行rebalance
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            try {
                //2 按照topic维度进行rebalance
                this.rebalanceByTopic(topic, isOrder);
            } catch (Throwable e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("rebalanceByTopic Exception", e);
                }
            }
        }
    }
 
    this.truncateMessageQueueNotMyTopic();
}
```

而Kafka与RocketMQ不同,Kafka是将所有Topic下的所有队列合并在一起,进行Rebalance.<br>


<br>
**<font size="4">单个Topic的Rebalance流程</font>** <br>

1. 获得Rebalance元数据信息
2. 进行队列分配
	> 这里分配策略使用AllocateMessageQueueStrategy接口表示,提供了多种实现
3. 分配结果处理


那么接下来回答本节开头提出的第一个问题,如何保证分配结果的一致性,其实是通过以下两个手段:
* 首先,在分配之前,需要对Topic下的多个队列进行排序,对多个消费者实例也按照id进行排序
* 其次,每个消费者需要使用相同的分配策略

这里以AllocateMessageQueueAveragely分配为例来进行说明.假设某个Topic有10个队列,消费者组有3个实例c1,c2,c3,使用AllocateMessageQueueAveragely分配结果如下图所示:
[![cU2fYV.png](https://z3.ax1x.com/2021/04/09/cU2fYV.png)](https://imgtu.com/i/cU2fYV)

在分配时,每个消费者(c1、c2、c3)平均分配3个,此时还多出1个,多出来的队列按顺序分配给消费者队列的头部元素,因此c1多分配1个,最终c1分配了4个队列.<br>

尽管每个消费者是各自给自己分配,但是因为使用的相同的分配策略,定位从队列列表中哪个位置开始给自己分配,给自己分配多少个队列,从而保证最终分配结果的一致.