---
layout:     post
title:      "RocketMQ 生产者如何发送消息"
date:       2021-04-07 23:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. MessageQueue](#jump1)
<br>
[二. 生产者发送消息的时候写入哪个MessageQueue](#jump2)
<br>
[三. 如果某个Broker出现故障怎么办](#jump3)
<br>








<br><br>
## <span id="jump1">一. MessageQueue</span>

在RocketMQ的工作台里,可以进行Topic的创建,在创建Topic的时候需要指定一个很关键的参数,就是MessageQueue,它代表了这个Topic对应了多少个队列,也就是多少个MessageQueue.
[![cGIGVI.png](https://z3.ax1x.com/2021/04/07/cGIGVI.png)](https://imgtu.com/i/cGIGVI)

所以其实MessageQueue就是RocketMQ中非常关键的一个数据分片机制,通过MessageQueue将一个Topic的数据拆分为了很多个数据分片,然后每个Broker机器上都存储了一部分MessageQueue,通过这个方法,实现了Topic数据的分布式存储.<br>



<br><br>
## <span id="jump2">二. 生产者发送消息的时候写入哪个MessageQueue</span>

首先,生产者在启动的时候会跟NameServer进行通信(运行过程中也会定时通信)以获取最新的Topic的路由数据信息.所以生产者从这份数据信息中就会知道,一个Topic下有几个MessageQueue,哪些MessageQueue在哪台Broker机器上,哪些MessageQueue在另外的Broker机器上.
[![cGTCnJ.png](https://z3.ax1x.com/2021/04/07/cGTCnJ.png)](https://imgtu.com/i/cGTCnJ)

然后生产者就会均匀的把消息写入各个MessageQueue中,当然,这里有各种负载均衡策略,此处只用均衡来举例.此时,假设单节点Broker能抗住7w并发,那么如果该Topic分布在两台Broker下,此Topic的写入也就可以达到14w的并发写入量了.以此,可以通过对分片的分散,使其分布在更多的Broker下来达到更高的写并发量<br>



<br><br>
## <span id="jump3">三. 如果某个Broker出现故障怎么办</span>

如果Master Broker挂了,此时正在等待的其他Slave Broker自动热切换为Master Broker,在切换完成前,这一组Broker就处于无法写入状态.<br>

此时如果还是按照之前的策略来把数据写入各个Broker上的MessageQueue,那么会导致在一段时间内,每次访问到这个挂掉的Master Broker都会访问失败,这明显不是我们想要的.<br>

对于这个问题,通常来说建议在Producer中开启一个开关,就是:sendLatencyFaultEnable.一旦打开了这个开关,那么他会有一个自动容错机制,比如某次访问一个Broker发现网络延迟有500ms,然后还无法访问,那么就会自动回避访问这个Broker一段时间,比如接下来的3000ms,都不会访问这个Broker了.这样的话,就可以避免一个Broker故障之后,短时间内生产者频繁的发送消息到这个故障的Broker上去,导致出现较多次数的异常.而是在一个Broker故障之后,自动回避一段时间不要访问这个Broker,过段时间再去访问它.这样一段时间之后,可能这个Master Broker已经恢复好了,或者它的Slave Broker成功切换为了Master Broker,此时就可以正常访问了.<br>