---
layout:     post
title:      "kafka 分区分配策略"
date:       2021-06-02 12:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---







## 导航
[一. 分区分配策略](#jump1)
<br>
[二. 消费者协调器和组协调器](#jump2)
<br>










<br><br>
## <span id="jump1">一. 分区分配策略</span>

客户端参数partition.assignment.strategy来配置消费者与主题之间的分区分配策略.kafka默认采取RangeAssignor策略<br>

这里首先放上两种case,便于说明后面各种分区分配策略的效果,首先假设消费组内有2个消费者C0和C1,都订阅了主题t0和t1,下面来看case:
* A : 每个主题有4个分区,即可标识为:t0p0,t0p1,t0p2,t0p3,t1p0,t1p1,t1p2,t1p3
* B : 每个主题有3个分区,即可标识为:t0p0,t0p1,t0p2,t1p0,t1p1,t1p2

<br>
**<font size="5">RangeAssignor ---- kafka默认策略</font>** <br>

RangeAssignor分配策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度,然后将分区按照跨度进行平均分配,以保证分区尽可能均匀地分配给所有的消费者.对于每一个主题,RangeAssignor策略会将消费组内所有订阅这个主题的消费者按照名称的字典排序,然后为每个消费者划分固定的分区范围,如果不够平均分配,那么字典序靠前的消费者会被多分配一个分区.<br>

该分配策略下,两种case的分配结果
* A : 
    > C0 : t0p0,t0p1,t1p0,t1p1 <br>
      C1 : t0p2,t0p3,t1p2,t1p3
* B :
    > C0 : t0p0,t0p1,t1p0,t1p1 <br>
      C1 : t0p2,t1p2

可见,上述策略在caseB情况下并不均匀.<br>


<br>
**<font size="5">RoundRobinAssignor</font>** <br>

RoundRobinAssignor 分配策略的原理是将消费组内所有消费者及消费者订阅的所有主题的分区按照字典序排序,然后通过轮询方式逐 个将分区依次分配给每个消费者.<br>

该分配策略下,两种case的分配结果
* A : 
    > C0 : t0p0,t0p2,t1p0,t1p2 <br>
      C1 : t0p1,t0p3,t1p1,t1p3
* B :
    > C0 : t0p0,t0p2,t1p1 <br>
      C1 : t0p1,t1p0,t1p2

可见该策略下,两种case都是分配均匀的,这种均匀体现在是整体 topic-partition 集合下的均匀,而不是每个topic下的分区都均匀.<br>

当然,这种场景也有个前提,就是消费者组内的消费者,订阅的topic都是相同的.如果组内消费者订阅的topic集合都不同,那就有还是会出现不均情况.<br>

当然,实际生产中,很少会有同一组内消费者,分别订阅不同topic的情况,既然有了分组,那就应该是让组内消费者都是同质的,否则就应该属于不同的组.<br>


<br>
**<font size="5">StickyAssignor</font>** <br>

这种分配策略主要有两个目的:
1. 分区的分配要尽可能均匀
2. 分区的分配尽可能与上次分配的保持相同

当两者发生冲突时,第一个目标优于第二个目标.这个分配策略要比前两种复杂的多,这里就先不深入挖掘了,只要了解其核心目标就可以了.<br>



<br><br>
## <span id="jump2">二. 消费者协调器和组协调器</span>

多个消费者之间的分区分配是需要协调的,这个协同过程都是交由消费者协调器(ConsumerCoordinator)和组协调器(GroupCoordinator)来完成的,它们之间使用一套组协调协议进行交互.<br>

kafka将全部消费组分成多个子集,每个消费组的子集在服务端对应一个GroupCoordinator对其进行管理,GroupCoordinator是kafka服务端中用于管理消费组的组件.而消费者客户端中的ConsumerCoordinator组件负责与GroupCoordinator进行交互.<br>

ConsumerCoordinator 与 GroupCoordinator 之间最重要的职责就是负责执行消费者再均衡的操作,包括前面提及的分区分配的工作也是在再均衡期间完成的.就目前而言,一共有如下几种情形会触发再均衡的操作:
* 有新的消费者加入消费组
* 有消费者宕机下线.消费者并不一定需要真正下线,例如遇到长时间GC,网络延迟导致消费者长时间未向GroupCoordinator发送心跳等情况时,GroupCoordinator会认为消费者已经下线
* 有消费者主动退出消费组
* 消费组所对应的GroupCoordinator节点发生了变更
* 消费组内订阅的任一主题或者主题的分区数量发生变化


<br>
**<font size="5">Rebalance流程</font>** <br>

1. 与消费者所属消费组的GroupCoordinator所在的broker建立连接(如果当前已有连接,就不用再建立了)
2. 进入加入消费组的阶段,在此阶段的消费者会向GroupCoordinator发送JoinGroupRequest请求,并处理响应.
    > 消费者在发送JoinGroupRequest请求之后会阻塞,等待kafka服务端的响应
3. GroupCoordinator内部逻辑
    * 选举消费组的leader,策略很随意,可以说是随机的
    * 选出一个分区分配策略
    * 发送JoinGroupResponse响应给各个消费者,响应中包括前面逻辑中计算出来的分区策略等等信息
4. 由被选举出来的leader消费者根据收到的响应中的分区策略实施分区分配,然后通过GroupCoordinator来转发同步分配方案
    > 具体点就是,每个消费者收到JoinGroupResponse后,会再次向GroupCoordinator发送SyncGroupRequest请求.而leader消费者会在该请求中附带上它计算出来的分配方案结果
5. GroupCoordinator向各个消费者返回SyncGroupResponse,其中携带分配结果
6. 消费者收到SyncGroupResponse后,执行重分配
7. 消费者恢复正常工作,恢复消费

这里说明一点,Rebalance期间,消费者是不消费消息的.<br>





