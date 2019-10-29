---
layout:     post
title:      "消息队列--RocketMQ Producer"
date:       2019-10-28 15:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---

##	RocketMQ架构

![KfgR3D.png](https://s2.ax1x.com/2019/10/29/KfgR3D.png)


##	NameServer

在Apache RocketMQ中,名称服务器旨在协调分布式系统的每个组件,并通过管理主题路由信息来履行大部分职责.

大致来说,管理包括两个部分:
*	各个Broker节点周期性的更新元数据信息,包括他们持有的topic到每个name server中,这些数据信息被保存在每一个name server中.
*	name servers服务器向client提供服务,包括producers和comsumers.给他们提供最新的路由信息.

因此,在启动Brokers和clients之前,需要启动name servers,并在Brokers和client启动时配置name servers的地址.


name servers提供轻量级的服务发现和路由.每一个name server都记录全部的路由信息,提供相应的读和写服务并**支持快速存储扩展(这句我没懂,怎么个扩展法?)**

<font color="red">注意,name server之间不通信.</font>

##	Broker

Broker通过提供主题和队列的轻量级机制,专注于消息存储.他们支持push和pull模式,包含容错机制(2个或3个副本),并提供强大的峰值填充功能(流量削峰)和按其原始时间顺序累积数千亿条消息的能力.此外,提供灾难恢复,丰富的指标统计信息和警报机制,而这是传统消息传递系统所没有的.

Broker启动的时候,会将自己在本地存储的话题配置(默认位于$HOME/store/config/topic.json目录)中的所有话题加载内存中去,然后会将这些所有的话题全部同步到所有的Name服务器中.同时,Broker也会启动一个定时任务,默认每隔30秒来执行一次话题全同步.

**因此,name server之间不需要通信,只依赖每个Broker节点向name server同步数据即可,即便name server间会有短暂的数据不一致,但不影响,数据会达到最终一致性.这样性能高.**

Broker与name server之间数据同步示意图:
![KfgBu9.png](https://s2.ax1x.com/2019/10/29/KfgBu9.png)

