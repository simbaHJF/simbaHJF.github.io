---
layout:     post
title:      "RocketMQ 事务消息"
date:       2021-04-11 15:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---





## 导航
[一. RocketMQ事务消息的实现原理](#jump1)
<br>
[二. half消息是如何对消费者不可见的](#jump2)
<br>









<br><br>
## <span id="jump1">一. RocketMQ事务消息的实现原理</span>

RocketMQ的事务消息,是基于半包消息实现的.

[![cwHn1J.png](https://z3.ax1x.com/2021/04/11/cwHn1J.png)](https://imgtu.com/i/cwHn1J)

这里的第4步中,如果是commit,那么Broker收到后会将这条半包消息进行commit;如果是rollback,那么Broker收到后会将这这条半包消息标记删除,注意这里不是真正删除,因为是顺序写的,随意随机定位到某一位置在清除文件这一行是很低效的,所以这里是通过一种标记来表示其删除.<br>


下面再来看下半包消息发送成功但未收到Broker返回的响应,或者给Broker发送commit/rollback消息失败的情况.

[![cwbaPU.png](https://z3.ax1x.com/2021/04/11/cwbaPU.png)](https://imgtu.com/i/cwbaPU)

RocketMQ会通过自己的补偿机制,扫描处于half状态的消息,如果一直没对这个消息进行commit/rollback操作,超过一定时间,它就会回调provider端的一个接口,以查询这个msg的状态,以便对这个半包消息作出commit或者是标记删除的操作.<br>



<br><br>
## <span id="jump2">二. RocketMQ对half消息的处理机制</span>

RocketMQ发现收到的是一个half消息时,它不会把这个half消息的offset写入对应Topic的ConsumeQueue中,因此这是该half消息对consumer端是不可见的,RocketMQ会把这条half消息写入到自己内部的"RMQ_SYS_TRANS_HALF_TOPIC"这个Topic对应的一个ConsumeQueue里去.<br>

如果后面收到了rollback或者通过自身的补偿机制扫描回调half发现其应该rollback,RocketMQ会用一个OP操作来标记half消息状态,RocketMQ内部有一个OP_TOPIC,此时会写一条rollback OP记录到这个Topic里,标记某个half消息时rollback了.<br>

另外这里还需要注意一点,在补偿机制中回调接口用以判断hafl消息状态,最多会回调15次,如果15次之后都没法告知RocketMQ这条half消息的状态,那么RocketMQ会自动将这条消息rollback.<br>

[![cwOP8U.png](https://z3.ax1x.com/2021/04/11/cwOP8U.png)](https://imgtu.com/i/cwOP8U)


下面再来看下如果执行commit操作,RocketMQ对half消息的处理.<br>

如果执行了commit操作,RocketMQ会在OP_TOPIC里写入一条记录,标记half消息已经是commit状态.接着就会把这条原本放在RMQ_SYS_TRANS_HALF_TOPIC中的hafl消息正常写入到其真正Topic对应的ConsumeQueue中,这样consumer端就可以消费到这条消息了.<br>


