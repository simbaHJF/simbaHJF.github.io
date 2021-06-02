---
layout:     post
title:      "kafka controller broker"
date:       2021-06-01 18:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---










## 导航
[一. 什么是Controller Broker](#jump1)
<br>
[二. Controller Broker是如何被选出来的](#jump2)
<br>












<br><br>
## <span id="jump1">一. 什么是Controller Broker</span>

在分布式系统中,通常需要有一个协调者,该协调者会在分布式系统发生异常时发挥特殊的作用.在Kafka中该协调者称之为控制器(Controller),其实该控制器并没有什么特殊之处,它本身也是一个普通的Broker,只不过需要负责一些额外的工作(追踪集群中的其他Broker,并在合适的时候处理新加入的和失败的Broker节点、分配新的leader分区,leader选举,向其他普通broker同步元数据信息等).值得注意的是 : Kafka集群中始终只有一个Controller Broker



<br><br>
## <span id="jump2">二. Controller Broker是如何被选出来的</span>

Broker 在启动时,会尝试去 ZooKeeper 中创建 /controller 临时节点.Kafka 当前选举控制器的规则是 : 第一个成功创建 /controller 节点的 Broker 会被指定为控制器.

临时节点存储内容参考如下:
``
{ "version" : 1 ,"brokerid" : 0 , "timestamp" : "1529210278988" }
``

为了避免脑裂,造成多个controller的情形出现,kafka还通过zk存储一个持久化节点 /controller_epoch,用于存储纪元号.



<br><br>
## <span id="jump3">三. 分区leader的选举</span>

这里先介绍kafka中三个副本集合概念:
* AR ---- Assigned Replicas 总的分配副本,一个分区的AR集合在分配的时候就被指定了,并且只要不发生重分配的情况,该集合内部副本的顺序是保持不变的.
* ISR ---- In-sync Replicas,与leader副本保持一定程度同步的副本
* OSR ---- Out-of-Sync Replicas,脱离同步副本

[![2u4Vyt.png](https://z3.ax1x.com/2021/06/01/2u4Vyt.png)](https://imgtu.com/i/2u4Vyt)

发生分区leader选举的场景:
* 创建分区(创建主题或增加分区)
* 分区下线(分区原leader下线)

leader选举和对应的选举策略有关,这里介绍一个基本的OftlinePartitionLeaderElectionStrategy:
1. 按照AR集合中副本的顺序查找第一个存活的副本,并且这个副本在ISR集合中
2. 将该副本提升为分区leader副本




