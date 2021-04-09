---
layout:     post
title:      "RocketMQ 消费者获取消息及ACK"
date:       2021-04-09 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. 消费组](#jump1)
<br>
[二. 消费模式](#jump2)
<br>
[三. 重温MessageQueue,CommitLog,ConsumeQueue之间的关系](#jump3)
<br>
[四. MessageQueue与消费者的关系](#jump4)
<br>
[五. Push模式 VS Pull模式](#jump5)
<br>
[六. Broker是如何将消息读取出来返回给Consumer的](#jump6)
<br>
[七. 消费者如何处理消息,进行ACK以及提交消费进度](#jump7)
<br>
[八. 消费组中出现机器宕机或者扩容加机器,会怎么处理](#jump8)
<br>






<br><br>
## <span id="jump1">一. 消费组</span>

[![cNKkb6.png](https://z3.ax1x.com/2021/04/09/cNKkb6.png)](https://imgtu.com/i/cNKkb6)

每个消费者组,都会消费到这个消息,一个消费者组内,只会消费一遍.<br>



<br><br>
## <span id="jump2">二. 消费模式</span>

<br>
**<font size="4">集群模式消费</font>** <br>

默认情况下,RocketMQ采用年集群模式消费,也就是说,一个消费组获取到一条消息,只会交给组内的一台机器去处理,不是每台机器都可以获取到这条消息的.


<br>
**<font size="4">广播模式消费</font>** <br>

可以通过如下设置来改变为广播模式消费:
```
consumer.setMessageModel(MessageModel.BROADCASTING);
```

修改为广播模式消费后,那么对于消费组获取到的一条消息,组内每台机器都可以获取到这条消息.但是相对而言,广播模式其实用的很少,常见基本上都是使用集群模式来进行消费的.<br>



<br><br>
## <span id="jump3">三. 重温MessageQueue,CommitLog,ConsumeQueue之间的关系</span>

[![cNJ9nH.png](https://z3.ax1x.com/2021/04/09/cNJ9nH.png)](https://imgtu.com/i/cNJ9nH)

Topic中的多个MessageQueue会分散在多个Broker上,在每个Broker机器上,一个MessageQueue就对应了一个ConsumeQueue,当然在物理磁盘上其实是对应了多个ConsumeQueue文件的,此处为了方便,逻辑上可以大致理解为一一对应关系.<br>

<font color="red">但是对于一个Broker机器而言,存储在它上面的所有Topic以及MessageQueue的消息数据都是写入一个统一的CommitLog的,如上图中CommitLog代表的是Broker上所有消息都是往里面写的.</font> <br>

对于Topic的各个MessageQueue而言,就是通过各个ConsumeQueue文件来存储属于MessageQueue的消息在CommitLog文件中的物理地址,就是一个offset偏移量.<br>



<br><br>
## <span id="jump4">四. MessageQueue与消费者的关系</span>

对于一个Topic上的多个MessageQueue,是如何由一个消费组中的多台机器来进行消费的呢?<br>

这里的源码实现算法是较为复杂的,但基本原理可以简单理解为:它会较为均匀的将MessageQueue分配给消费组的多台机器来消费.<br>

举个例子,假设某个Topic下有4个MessageQueue,这4个MessageQueue分布在两个Master Broker上,每个Master Broker上有两个MessageQueue.然后某个消费组里面有两台机器,那么一般正常情况下,就是让这两台机器分别负责两个MessageQueue的消费.<br>

所以,大致可以认为一个Topic的多个MessageQueue会均匀分摊给消费组内的多个机器去消费,这里的一个原则就是:**<font color="red">一个MessageQueue只能被一个消费机器去处理,但是一台消费者机器可以负责多个MessageQueue的消息处理.</font>**<br>



<br><br>
## <span id="jump5">五. Push模式 VS Pull模式</span>

在RocketMQ里,Consumer消费消息被分为两类:Push 与 Pull.对应 MQPushConsumer 和 MQPullConsumer.<font color="red">但是其实二者的本质都是拉模式(Pull),即consumer轮询从Broker拉取消息</font>.<br>

二者的区别是:
* push方式里,Consumer把轮询过程封装了,并注册MessageListener监听器,收取到消息后,唤醒MessageListener的consumeMessage()来消费,对用户而言,感觉像是被推送过来的.
* pull方法里,取消息的过程需要用户自己写,首先通过要消费的Topic拿到MessageQueue的集合,遍历MessageQueue集合,然后针对每个MessageQueue批量取消息,一次取完后,记录该队列下一次要取的开始offset,知道取完,再换另一个MessageQueue.

Pull模式的代码类似如下:
```
public class PullConsumer {
    private static final Map<MessageQueue, Long> offseTable = new HashMap<MessageQueue, Long>();
 
 
    public static void main(String[] args) throws MQClientException {
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name_5");
        consumer.setNamesrvAddr("192.168.0.104:9876");
        consumer.start();
 
        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest");
        for (MessageQueue mq : mqs) {
            System.out.println("Consume from the queue: " + mq);
            SINGLE_MQ: while (true) {
                try {
                    PullResult pullResult =
                            consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                    System.out.println(pullResult);
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset());
                    switch (pullResult.getPullStatus()) {
                    case FOUND:
                        // TODO
                        break;
                    case NO_MATCHED_MSG:
                        break;
                    case NO_NEW_MSG:
                        break SINGLE_MQ;
                    case OFFSET_ILLEGAL:
                        break;
                    default:
                        break;
                    }
                }
                catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        consumer.shutdown();
    }
 
    private static void putMessageQueueOffset(MessageQueue mq, long offset) {
        offseTable.put(mq, offset);
    }
 
    private static long getMessageQueueOffset(MessageQueue mq) {
        Long offset = offseTable.get(mq);
        if (offset != null)
            return offset; 
        return 0;
    }
}
```

此外,拉取消息时,涉及到一个请求挂起和长轮询机制,当请求发送到Broker,结果Broker发现没有新的消息时,就会让请求线程挂起,默认是挂起15秒,然后这个期间会有后台线程每隔一会儿就去检查一下是否有新的消息,如果在这个挂起过程中,有新的消息到达了,会主动唤醒挂起的线程,然后把消息返回给Consumer.<br>

这里小结一下,为什么底层都是基于Pull模式的,为什么还要整出Push模式与Pull模式两种呢?<br>

我觉得是因为,Push模式下,框架为用户封装了很多底层操作,用起来更方便,但是个性化逻辑就较难实现.而如果用户想实现自己比较定制化的消息拉取消费逻辑,就可以采用Pull模式,比如说,举个例子:用户想每拉取完休息5秒,或者奇数分钟内拉取消息,偶数分钟内不拉取.这种比较特殊的动作,就可以通过Pull模式自己灵活实现.<br>



<br><br>
## <span id="jump6">六. Broker是如何将消息读取出来返回给Consumer的</span>

消费消息的本质,其实就是根据消费者要消费的MessageQueue以及开始消费的位置,去找对应的ConsumeQueue读取里面对应位置的消息在CommitLog中的物理offset偏移量,然后到CommitLog中根据offset读取消息数据,返回给消费者机器.<br>



<br><br>
## <span id="jump7">七. 消费者如何处理消息,进行ACK以及提交消费进度</span>

当消费者处理完一批消息后,消费者返回ACK同时会提交目前的一个消费进度到Broker上去,然后Broker会存储该消费组的消费进度,那么下次这个消费组只要再次拉取这个ConsumeQueue的消息,就可以从Broker记录的消费位置开始继续拉取,不用重头拉取了.<br>



<br><br>
## <span id="jump8">八. 消费组中出现机器宕机或者扩容加机器,会怎么处理</span>

这个时候,会进入一个rebalance环节,重新给各个消费机器分配他们要处理的MessageQueue.<br>

RocketMQ的rebalance触发是在consumer端完成的,也就是说,consumer端感知到Topic下的MessageQueue信息有变化或者消费组机器数量有变化(或者是NameServer通知的,或者是consumer端定时拉取NameServer中Topic下最新元信息),按照最新的MessageQueue数量信息进行客户端rebalance.<br>