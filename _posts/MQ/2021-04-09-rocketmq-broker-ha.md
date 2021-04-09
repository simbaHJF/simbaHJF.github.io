---
layout:     post
title:      "RocketMQ 基于DLedger技术的Broker主从同步原理"
date:       2021-04-09 10:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---





## 导航
[一. Broker高可用架构架构原理](#jump1)
<br>
[二. 基于DLedger技术替换Broker的CommitLog](#jump2)
<br>
[三. DLedger选举](#jump3)
<br>
[四. DLedger如何基于Raft协议进行多副本同步](#jump4)
<br>
[五. Leader Broker的崩溃恢复](#jump5)
<br>











<br><br>
## <span id="jump1">一. Broker高可用架构架构原理</span>

要Broker实现高可用,必须有一个Broker组,里面有一个是Leader Broker可以写入数据,然后让Leader Broker接收到数据之后,直接把数据同步给其他的Follower Broker.这样,一条数据会在Follower Broker上存有副本,此时如果Leader Broker宕机,那么就可以通过自动将其他Follower Broker切换为新的Leader Broker,继续接受客户端的数据写入.
[![ctohxf.png](https://z3.ax1x.com/2021/04/09/ctohxf.png)](https://imgtu.com/i/ctohxf)



<br><br>
## <span id="jump2">二. 基于DLedger技术替换Broker的CommitLog</span>

Broker的高可用架构就是基于DLedger技术来实现的.DLedger技术实际上自己有一个CommitLog机制,对写入的数据,它会写入自己机制下的CommitLog磁盘文件里去.所以,如果是基于DLedger来实现Broker高可用架构,实际上就是用DLedger先替换掉原来Broker自己管理的CommitLog,由DLedger来管理CommitLog,然后Broker还是基于DLedger管理的CommitLog去构建出来各个ConsumeQueue磁盘文件.
[![ctHeVe.png](https://z3.ax1x.com/2021/04/09/ctHeVe.png)](https://imgtu.com/i/ctHeVe)



<br><br>
## <span id="jump3">三. DLedger选举</span>

参见Raft



<br><br>
## <span id="jump4">四. DLedger如何基于Raft协议进行多副本同步</span>

数据同步分为两个阶段,一个是uncommitted阶段,一个是committed阶段.<br>

首先Leader Broker上的DLedger收到一条数据之后,会标记为uncommitted状态,然后他会通过自己的DLedgerServer组件,将这个uncommitted数据发送给Follower Broker的DledgerServer组件.<br>

接着Follower Broker的DLedgerServer收到uncommitted消息之后,返回一个ack给Leader Broker的DledgerServer,然后如果Leader Broker收到超过半数的Follower Broker返回的ack,就会将消息标记为committed状态.然后Leader Broker上的DLedgerServer就会发送committed消息给Follower Broker的DLedgerServer,让他们也把消息标记为committed状态.<br>

这里有一点需要注意,在Leader Broker收到生产者发来的消息时,同步给Follower Broker.之后,只要有超过半数的Follower Broker都写入了**<font color="red">uncommitted</font>**消息之后,Leader Broker就可以给生产者Producer端返回ACK了.哪怕此时,在Leader Broker将这条消息commit之前宕机了,超过半数的Follower Broker上也是有这条消息的,只不过是uncommitted状态,之后新选举出来的Leader Broker可以根据剩余Follower Broker上这个消息的状态去进行数据恢复,比如把消息状态调整为committed.<br>



<br><br>
## <span id="jump5">五. Leader Broker的崩溃恢复</span>

如果Leader Broker挂了,此时剩下的两个Follower Broker就会重新发起选举,他们会基于DLedger组件采用Raft协议的算法,选举出一个新的Leader Broker继续对外提供服务,而且会对没有完成的数据同步进行一些恢复性的操作,保证数据不会丢失.<br>



