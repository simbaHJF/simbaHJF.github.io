---
layout:     post
title:      "消息队列--RocketMQ Broker接受消息"
date:       2019-11-11 20:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---


#	Broker启动

首先,每台broker节点的数据存储,有固定的目录地址,记为broker数据的根目录.该路径为:用户家目录/store.(<font color="red">后面为了说明方便,将这个路径简单记为brokerRootDir</font>)

在BrokerStartup主类下,首先会调用BrokerStartup#createBrokerController(String[] args)方法,创建一个BrokerController,创建完成后,会执行controller.initialize()方法,对其进行初始化操作.broker的启动,基本是在这里完成的.下面,跟着controller.initialize()内部的实现逻辑,具体分析下:

##	this.consumerOffsetManager.load()

从名字上可以看出,这个方法是加载consumer的消息处理offset的.

首先,会去获取consumer消费offset的记录文件.这个文件是brokerRootDir/config/consumerOffset.json

然后,将其decode并反序列化为ConsumerOffsetManager,其内部主要通过一个map结构记录信息,key为topic@group,value的map中,key为具体哪条消息队列,value为offset.然后使用ConsumerOffsetManager内部的offsetTable属性来持有这个offsetTable.

```
public class ConsumerOffsetManager extends ConfigManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
    private static final String TOPIC_GROUP_SEPARATOR = "@";

    private ConcurrentMap<String/* topic@group */, ConcurrentMap<Integer, Long>> offsetTable =
        new ConcurrentHashMap<String, ConcurrentMap<Integer, Long>>(512);
	........
}
```

##	this.subscriptionGroupManager.load()

这个方法是加载订阅组信息文件.这个文件是brokerRootDir/config/subscriptionGroup.json

同样,首先读入文件,然后decode,然后反序列化.反序列化成的对象为SubscriptionGroupManager.其内部通过map存储信息,其中key为groupName,

```
private final ConcurrentMap<String, SubscriptionGroupConfig> subscriptionGroupTable =
        new ConcurrentHashMap<String, SubscriptionGroupConfig>(1024);
```

##	this.consumerFilterManager.load()

文件路劲:brokerRootDir/config/consumerFilter.json


##	this.messageStore.load()

前面三个文件都加载成功后,会执行这个方法
