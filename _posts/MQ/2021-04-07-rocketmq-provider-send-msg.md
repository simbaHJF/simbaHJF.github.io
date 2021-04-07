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