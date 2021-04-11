---
layout:     post
title:      "RocketMQ 重试队列与死信队列"
date:       2021-04-11 16:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---






## 导航
[一. 场景描述](#jump1)
<br>
[二. 重试队列](#jump2)
<br>
[三. 死信队列](#jump3)
<br>
[四. 特别注意](#jump4)












<br><br>
## <span id="jump1">一. 场景描述</span>

当consumer端系统数据库异常时,此时消费消息后无法在db中做相关记录,那么此时不能算做消息消费成功,也就不能给Broker返回CONSUME_SUCCESS,而此时需要返回的是RECONSUME_LATER,表示暂时无法完成这批消息的处理,让Broker稍后再把这批消息给到consumer进行消费.<br>



<br><br>
## <span id="jump2">二. 重试队列</span>

RocketMQ会有一个针对这个ConsumerGroup的重试队列,当RocketMQ收到返回的RECONSUME_LATER状态之后,会把这批消息放到这个消费组的重试队列中去,然后过一段时间之后,重试队列中的消息会再次给consumer,让其进行处理.如果再次失败,又返回了RECONSUME_LATER,那么会再过一段时间再让consumer来进行处理,默认最多是重试16次,每次重试之间的间隔时间是不一样的,可配,也可按默认.<br>

[![cwzq2V.png](https://z3.ax1x.com/2021/04/11/cwzq2V.png)](https://imgtu.com/i/cwzq2V)



<br><br>
## <span id="jump3">三. 死信队列</span>

当重试16次还是没有消费成功后,这时候就需要另外一个队列了,也即<font color="red">死信队列</font> <br>

重试16次还是一直没有成功后,这批消息会进入到死信队列.这是或者进行人工处理,或者专门开一个后台线程,订阅死信队列,对其中的消息不断的重试.<br>

[![c0S5QK.png](https://z3.ax1x.com/2021/04/11/c0S5QK.png)](https://imgtu.com/i/c0S5QK)



<br><br>
## <span id="jump4">四. 特别注意--有序性场景问题</span>

需要对消息做有序性保证的场景中,除了确保消息发送的有序性,同一key对应的消息路由到统一MessageQueue外,还有一点需要特别注意.<br>

那就是Consumer处理消息的时候,如果底层存储挂了导致消息处理失败,这是不能再返回RECONSUMER_LATER状态了,因为这样的话会造成稍后进行重试,而此时恰好底层存储好了,下一批消息处理成功了,而上一批需要重试的消息还要稍后才会发过来,这里就又会造成有序性问题了.<br>

所以,对于这种场景下,处理失败的消息,必须返回SUSPEND_CURRENT_QUEUE_A_MOMENT这个状态,意思是先等一会儿,一会儿再继续处理这批消息,而不能把这批消息放入重试队列里去而转头直接处理下一批消息.<br>

