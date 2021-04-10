---
layout:     post
title:      "RocketMQ 消息可靠性保证--消息不丢失"
date:       2021-04-10 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. Provider端消息投递阶段的安全性保证](#jump1)
<br>
[二. Broker端消息存储阶段的安全性保证](#jump2)
<br>
[三. Consumer端消息消费阶段的安全性保证](#jump3)
<br>









<br><br>
## <span id="jump1">一. Provider端消息投递阶段的安全性保证</span>

Provider端进行投递消息时,存在三种模式:
* 同步模式
* 异步模式
* oneway模式

<br>
**<font size="4">同步模式下的消息可靠性保证</font>** <br>

同步发送时,provider端阻塞线程进行等待,直到服务器返回发送结果,结果状态为成功则消息已可靠投递给Broker;结果为失败或超时,则会进行重试,以确保消息可靠投递.因此这里的可靠性是 At Least Once,但有可能重复发送.<br>


<br>
**<font size="4">异步模式下的消息可靠性保证</font>** <br>

需要在消息发送的回调接口实现中,做相应处理.如果返回的结果是成功,那么消息已经可靠投递,如果返回的结果是失败或超时,可将消息存入失败队列或缓存或文件或db中,通过后置的定时任务和重试逻辑进行重新发送,以确保最终能够完成可靠投递.<br>


<br>
**<font size="4">oneway模式下的消息可靠性保证</font>** <br>

该模式下,无法保证消息的可靠投递



<br><br>
## <span id="jump2">二. Broker端消息存储阶段的安全性保证</span>

Broker端会在以下场景下面临消息可靠性问题:
1. Broker正常关闭
2. Broker宕机
3. Broker节点机器设备损坏

对于1,没有任何问题,所有消息都可以保证可靠性.<br>

对于2,同步刷盘策略下可以保证数据可靠性,异步刷盘可能导致少量数据丢失.<br>

对于3,属于单点故障,可通过master-slave架构做副本HA的高可用方案,不过此时又涉及到主从同步机制的问题.同步复制时(消息投递到master broker后,同步给slave,同步完成后才向provider端返回ack)可保证可靠性,但性能降低;异步复制时(slave发送请求主动向master拉取进行同步)会有少量消息丢失情况.<br>



<br><br>
## <span id="jump3">三. Consumer端消息消费阶段的安全性保证</span>

首先,由于存在重复投递,也即会造成重复消费的问题,所以,consumer端首先要保证消费的幂等性.<br>

其次,采用先消费,消费成功后再提交ack和offset的方式,确保消息消费的可靠性.<br>