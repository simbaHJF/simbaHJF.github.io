---
layout:     post
title:      "消息队列--RocketMQ Producer的启动过程"
date:       2019-10-28 15:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---

> 参考:<br>
极客时间--李玥--消息队列高手课<br>
赵坤的个人网站:https://www.kunzhao.org/blog/2018/03/02/rocketmq-message-send-flow/

##	RocketMQ架构

![KfgR3D.png](https://s2.ax1x.com/2019/10/29/KfgR3D.png)


##	NameServer

在Apache RocketMQ中,名称服务器旨在协调分布式系统的每个组件,并通过管理主题路由信息来履行大部分职责.

大致来说,管理包括两个部分:
*	各个Broker节点周期性的更新元数据信息,包括他们持有的topic到每个name server中,这些数据信息被保存在每一个name server中.
*	name servers服务器向client提供服务,包括producers和comsumers.给他们提供最新的路由信息.

因此,在启动Brokers和clients之前,需要启动name servers,并在Brokers和client启动时配置name servers的地址.


name servers提供轻量级的服务发现和路由.每一个name server都记录全部的路由信息,提供相应的读和写服务并支持快速存储扩展

<font color="red">注意,name server之间不通信.</font>

##	Broker

Broker通过提供主题和队列的轻量级机制,专注于消息存储.他们支持push和pull模式,包含容错机制(2个或3个副本),并提供强大的峰值填充功能(流量削峰)和按其原始时间顺序累积数千亿条消息的能力.此外,提供灾难恢复,丰富的指标统计信息和警报机制,而这是传统消息传递系统所没有的.

Broker启动的时候,会将自己在本地存储的话题配置(默认位于$HOME/store/config/topic.json目录)中的所有话题加载内存中去,然后会将这些所有的话题全部同步到所有的Name服务器中.同时,Broker也会启动一个定时任务,默认每隔30秒来执行一次话题全同步.

**因此,name server之间不需要通信,只依赖每个Broker节点向name server同步数据即可,即便name server间会有短暂的数据不一致,但不影响,数据会达到最终一致性.这样性能高.**

Broker与name server之间数据同步示意图:
![KfgBu9.png](https://s2.ax1x.com/2019/10/29/KfgBu9.png)


##	Producer的启动

```
public void init() throws Exception {
        String producerGroupTemp = producerGroupPrefix + System.currentTimeMillis();
        producer = new DefaultMQProducer(producerGroupTemp);
        producer.setNamesrvAddr("127.0.0.1:9876");
        
        ....

        producer.start();

        ....
    }
```

producer.start()方法即是producer的启动方法,方法内部会调用其内部对象DefaultMQProducerImpl实例的start(final boolean startFactory)方法进行启动.

```
public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;

                this.checkConfig();

                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }

                // 获取MQClientInstance的实例mQClientFactory，没有则自动创建新的实例
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

                // 在mQClientFactory中注册自己
                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

                // 启动mQClientFactory
                if (startFactory) {
                    mQClientFactory.start();
                }

                log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                    this.defaultMQProducer.isSendMessageWithVIPChannel());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The producer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }

        // 给所有Broker发送心跳
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    }
```
这个方法中,使用serviceState来记录和管理自身的服务状态,是一种状态模式的变种.启动过程大体如下:
1.	通过单例模式的MQClientManager获取MQClientInstance的实例mQClientFactory,没有则自动创建新的实例
2.	在mQClientFactory中注册自己
3.	启动mQClientFactory
4.	给所有Broker发送心跳

实例mQClientFactory对应的类MQClientInstance是RocketMQ客户端的顶层类,大多数情况下,可以简单理解为每个客户端对应类MQClientInstance的一个实例.这个实例维护着客户端的大部分状态信息,以及所有的Producer,Consumer和各种服务的实例.

进一步看一下MQClientInstance#start()方法:
```
public void start() throws MQClientException {

        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    this.startScheduledTask();
                    // Start pull service
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case RUNNING:
                    break;
                case SHUTDOWN_ALREADY:
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }
```

流程大致如下:
1.	启动实例mQClientAPIImpl,其中mQClientAPIImpl是类MQClientAPIImpl的实例,内部封装了netty,实现与其他组件的通信.
2.	启动各种定时任务
3.	启动拉取消息服务
4.	启动Rebalance服务
5.	启动Producer服务

在上述2中启动的定时任务主要有:从NameServer获取topic路由信息,清除下线Broker信息,向Broker发送心跳.

其中,从NameServer获取topic路由信息任务,只与某一个nameserver进行通信(如果这个链接正常的话,否则会销毁channel,换另一个nameserver建立连接).当然,Producer刚刚初始化的时候,这时还没有topic,只有一个

向Broker发送心跳的任务是给每个Broker都发送.



Producer启动流程中,有几个关键类:
1.	DefaultMQProducerImpl:	Producer的内部封装的类,大部分Producer的业务逻辑,都是通过DefaultMQProducerImpl来实现的.DefaultMQProducerImpl为一种门面模式设计.
2.	MQClientInstance:	这个类中封装了客户端一些通用的业务逻辑.
3.	MQClientAPIImpl:	这个类中封装了客户端服务端的RPC,对调用者隐藏真正的网络通信部分的具体实现.
4.	NettyRemotingClient:	RocketMQ各进程之间网络通信的底层实现类.


