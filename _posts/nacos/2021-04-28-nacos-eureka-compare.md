---
layout:     post
title:      "Nacos与Eureka作为注册中心的一些对比"
date:       2021-04-28 18:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - nacos

---






## 导航
[一. CP还是AP](#jump1)
<br>
[二. server端服务元数据的存储和变更](#jump2)
<br>









<br><br>
## <span id="jump1">一. CP还是AP</span>

<br>
**<font size="5">Eureka</font>** <br>

Eureka是AP的,每个server节点间都是对等的peer,因此client连接哪台server都是可以的,server之间会同步服务实例的心跳以及其他一些元数据信息,以此来完成各个server节点间的最终一致性.但同步各服务实例的心跳数据也是一项重量级的工作,尤其在集群规模达到一定程度时,这会达到一个庞大的通信规模,因此Eureka集群中服务实例数量较大时,较为容易出现瓶颈.<br>


<br>
**<font size="5">Nacos</font>** <br>

Nacos同时支持CP与AP.
> 当注册实例是persistent类型(持久类型)时,Nacos采用CP模式,通过其自己内部对raft算法的一套简单实现JRaft来达成CP模式,在注册实例时,对于发送到Follower节点的写数据请求,Follower会将其转发给Leader节点处理,然后Leader依据raft算法思想,完成数据写入和提交,但nacos这里没有完整的实现两阶段提交的思想,只有一个commit阶段,同时通过异步http+CountDownLatch来完成过半提交判断

> 当注册实例是ephemeral类型(临时类型,默认)时,Nacos采用AP模式,通过其自己内部实现的一套叫Distro协议来达成AP模式.

<font color="red">这里需要指出一项在AP模式下Nacos与Eureka的不同,nacos-client同样也是连接到某一个server节点上,当有新服务实例注册或者某实例下线时,server会将信息同步给其他server,而server不需要将心跳信息等同步给其他节点,为什么可以做到不同步心跳呢?这是因为Nacos的server端通过DistroFilter实现了对groupedServiceName的哈希处理,以此实现了服务的分配,当心跳提交到某一server,而server判断该服务实例不由自己处理时,会将其转发到对应的server节点,这就完美解决了Eureka中集群规模增大后的巨量心跳数据同步问题.</font> <br>

但这种设计也有其局限性,比如在server某节点宕机时,那么原本的哈希分配结果就会被打乱,需要重新hash.



<br><br>
## <span id="jump2">二. server端服务元数据的存储和变更</span>

<br>
**<font size="5">Eureka</font>** <br>

采用了锁机制,而为了降低并发竞争锁问题,通过三级缓存+定时更新的方式实现服务信息的存储与变更,也正是因为这种设计模式,不难发现,其服务发现机制(实例上线与下线)是不太及时的,生产环境下不可能采用默认的那套心跳时长设置以及三级缓存定时时长设置,否则在服务信息变更时会有很大问题.<br>

同时,client对服务状态变更的感知,只依赖于定时的主动pull,因此时效性更是差.<br>


<br>
**<font size="5">Nacos</font>** <br>

Nacos采用了'异步+队列+事件通知+copyOnWrite(空间换时间)'的机制来实现服务信息缓存与更新,没有三级缓存,只有一级缓存,因此服务实例状态变更的及时性要比Eureka好的多.<br>

另外,Nacos不只依靠client定时向server拉取服务信息,server同样会通过udp向client及时推送服务信息的变更.<br>