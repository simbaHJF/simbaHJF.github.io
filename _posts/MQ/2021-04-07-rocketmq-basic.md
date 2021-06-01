---
layout:     post
title:      "RocketMQ 基本架构"
date:       2021-04-07 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---



## 导航
[一. RocketMQ 核心模块](#jump1)
<br>
[二. 如果Broker挂了,系统是怎么感知到的](#jump2)
<br>
[三. Master Broker与Slave Broker之间的消息同步](#jump3)
<br>
[四. RocketMQ 实现读写分离了吗](#jump4)
<br>
[五. RocketMQ Master Broker挂掉了怎么办](#jump5)
<br>
[六. 向RocketMQ发送消息的几种方式](#jump6)
<br>
[七. 从RocketMQ消费消息的几种方式](#jump7)
<br>









<br><br>
## <span id="jump1">一. RocketMQ 核心模块基本原理</span>

* NameServer, 负责管理集群中所有Broker的信息. 部署方式上,采用集群部署方式,以做到高可用性.NameServer集群中每台机器之间都是无状态对等节点,存储全部Broker集群信息,彼此之间没有通信关系.
* Broker集群, 主从架构多副本, 用于实现MQ的核心功能. 每个Broker(包括master和slave)启动时,需要向所有的NameServer进行注册.Broker向每个NameServer建立TCP长连接
* 生产者系统, 向MQ中生产消息的业务系统. 定时发送请求到NameServer去拉取最新的集群Broker信息.
* 消费者系统, 通过MQ来消费消息的业务系统. 定时发送请求到NameServer去拉取最新的集群Broker信息



<br><br>
## <span id="jump2">二. 如果Broker挂了,系统是怎么感知到的</span>

Broker和NameServer之间建立心跳机制,Broker定时给所有的NameServer发送心跳,告诉每个NameServer自己当前处于健康存活状态.Broker跟NameServer之间的通信是基于长连接的,即Broker会跟每个NameServer都建立一个TCP长连接,然后定时通过TCP长连接发送心跳请求,发送的心跳信息中还会带上自己这个Broker节点的一些其他元信息(包括自己存放了哪些Topic等)<br>

NameServer每次收到一个Broker的心跳,就更新一下该Broker的最近一次心跳时间.然后NameServer会每隔10秒进行一次检查,检查各个Broker的最近一次心跳时间,如果某个Broker超过120s都没发送心跳了,那么就认为这个Broker已经挂掉了. <br>

对于provider和consumer,则是通过定时主动请求NameServer拉取broker信息来感知的,因此有一定的延迟.<br>


<br><br>
## <span id="jump3">三. Master Broker与Slave Broker之间的消息同步</span>

* Master-Slave异步复制模式,采用的是Slave Broker不停的发送请求到Master Broker去拉取消息,即Pull模式.
	* 优点: 性能高
	* 缺点: Master宕机,有少量消息丢失(Slave还没将最新消息拉取过来)
* Master-Slave同步双写模式,消息写入时,主备都写成功,才向应用返回成功
	* 优点: 消息不会丢
	* 缺点: 性能有一定损失



<br><br>
## <span id="jump4">四. RocketMQ 实现读写分离了吗</span>

生产者写入消息,肯定是选择Master Broker去写入的.<br>

消费者端消费消息时,有可能从Master Broker拉取消息,也有可能从Slave Broker拉取消息.<br>

消费者系统在获取消息的时候会先发送请求到Master Broker上去,请求获取一批消息,此时Master Broker返回消息给消费者系统的时候,会根据当时Master Broker的负载情况和Slave Broker的同步情况,向消费者系统建议下一次拉取消息的时候是从Master Broker拉取还是从Slave Broker拉取.<br>

举例来讲,如果Master的负载很重,且此时要拉取的消息已经被Slave同步过去了,此时Master就会建议去从Slave拉取;而如果是另一种情况,Master上已经写入了100w消息而Slave同步很慢,只同步了90w消息,此时消费者系统可能已经拉取过95w消息了,那么此时肯定不能从Slave拉取,只能从Master拉取.<br>



<br><br>
## <span id="jump5">五. RocketMQ Master Broker挂掉了怎么办</span>

对消息的写入和获取都有一定的影响.<br>

在故障转移方面:
* RocketMQ 4.5版本之前,Master一旦故障,Slave是没办法自动切换成Master的,所以这种情况下,需要手动做运维操作,修改Slave的一些配置,重启Slave将其调整为Master.这会导致中间一段时间不可用,同时如果是在Master-Slave异步复制模式下,会丢失一些消息.因此4.5版本之前不是真正的高可用,因为它无法自动进行故障转移.
* RocketMQ 4.5版本之后,提供了一种新机制:Dledger,基于Raft算法.此时一旦Master Broker宕机了,可以通过Dledger进行leader选举,将一个Slave Broker选举为新的Master Broker,然后这个Master Broker就可以对外提供服务了.以此来完成自动的故障转移机制.

<font color="red">简单讲就是,Broker节点开启Dledger机制后,会通过Dledger相关机制,而自动具备了Master Broker故障时通过Raft进行leader选举从而将某个Slave Broker提升为Master的能力,实现自动的故障转移</font><br>

选举投票的范围是在这个broker下的master-slave架构内部进行的.举例来讲,broker-a有三个节点,id分别为1,2,3. <br>

broker-a-1当前为leader,broker-a-2和broker-a-3位follower,某刻leader也就是broker-a-1挂了,那么这个小集群内就剩2和3了,它俩内部之间会发起选举从而选出一个称为新的leader.<br>

这里不同于redis的sentinel模式下,先由sentinel集群内部选出一个leader sentinel,用它来完成slave提升为master的操作.也不同于cluster架构下,某个槽的主节点挂了,其下的slave向其他负责处理槽的master发起投票请求.<br>



<br><br>
## <span id="jump6">六. 向RocketMQ发送消息的几种方式</span>

* 同步发送
* 异步发送,设置成功和失败的回调函数
* oneway,执行一次发送,不管发送结果是成功还是失败都继续往下走,简单讲就是发完就过,不管结果



<br><br>
## <span id="jump7">七. 从RocketMQ消费消息的几种方式</span>

* Pull方式,consumer端从Broker拉取消息
* Push方式,给用户的感觉上像是Broker向Consumer推送消息,但底层实际上也是拉取消息