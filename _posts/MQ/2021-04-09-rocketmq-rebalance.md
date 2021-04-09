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