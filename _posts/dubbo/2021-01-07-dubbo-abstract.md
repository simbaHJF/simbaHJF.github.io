---
layout:     post
title:      "Dubbo概要"
date:       2021-01-07 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---






## 导航
一. [Dubbo架构](#jump1)
<br>
二. [Dubbo源码模块](#jump2)


## <span id="jump1">一. Dubbo架构</span>

Dubbo 核心架构图:

[![seciIU.jpg](https://s3.ax1x.com/2021/01/07/seciIU.jpg)](https://imgchr.com/i/seciIU)

**Registry:注册中心.** 负责服务地址的注册与查找,服务的 Provider 和 Consumer 只在启动时与注册中心交互.注册中心通过长连接感知 Provider 和 Consumer,在 Provider 出现宕机的时候,注册中心会立即通过与 Consumer 的长连接推送相关事件通知 Consumer.<br>

**Provider:服务提供者.** 在它启动的时候,会向 Registry 进行注册操作,将自己服务的地址和相关配置信息封装成 URL 添加到 Registry 中.<br>

**Consumer:服务消费者.** 在它启动的时候,会向 Registry 进行订阅操作.以 ZooKeeper 为例,订阅操作会从 ZooKeeper 中获取 Provider 注册的 URL,并在 ZooKeeper 中添加相应的监听器.如果 Provider URL 发生变更,Consumer 将会通过之前订阅过程中在注册中心添加的监听器,获取到最新的 Provider URL 信息,进行相应的调整,比如断开与宕机 Provider 的连接,并与新的 Provider 建立连接.Consumer 与 Provider 建立的是长连接,且 Consumer 会缓存 Provider 信息,所以一旦连接建立,即使注册中心宕机,也不会影响已运行的 Provider 和 Consumer.Consumer 请求 Provider时 ,从缓存的提供者地址列表中,基于软负载均衡算法,选一台提供者进行调用,如果调用失败,再选另一台调用.<br>

**Monitor:监控中心.** 用于统计服务的调用次数和调用时间.Provider 和 Consumer 在运行过程中,会在内存中统计调用次数和调用时间,定时发送统计数据到监控中心.监控中心在上面的架构图中并不是必要角色,监控中心宕机或不启动不会影响 Provider,Consumer 以及 Registry 的功能,只会丢失监控数据而已.<br>

**Container:服务运行容器.** 服务容器负责启动,加载,运行服务提供者.<br>


## <span id="jump2">二. Dubbo源码模块</span>

Dubbo 源码模块:

[![sefZbn.png](https://s3.ax1x.com/2021/01/07/sefZbn.png)](https://imgchr.com/i/sefZbn)

**dubbo-common 模块:** Dubbo 的一个公共模块,其中有很多工具类以及公共逻辑,如 Dubbo SPI 实现、时间轮实现、动态编译器等.<br>

**dubbo-remoting 模块:** Dubbo 的远程通信模块,其中的子模块依赖各种开源组件实现远程通信.在 dubbo-remoting-api 子模块中定义该模块的抽象概念,在其他子模块中依赖其他开源组件进行实现,例如,dubbo-remoting-netty4 子模块依赖 Netty 4 实现远程通信,dubbo-remoting-zookeeper 通过 Apache Curator 实现与 ZooKeeper 集群的交互.<br>

**dubbo-rpc 模块:** Dubbo 中对远程调用协议进行抽象的模块,其中抽象了各种协议,依赖于 dubbo-remoting 模块的远程调用功能.dubbo-rpc-api 子模块是核心抽象,其他子模块是针对具体协议的实现,例如,dubbo-rpc-dubbo 子模块是对 Dubbo 协议的实现,依赖了 dubbo-remoting-netty4 等 dubbo-remoting 子模块. dubbo-rpc 模块的实现中只包含一对一的调用,不关心集群的相关内容.<br>

**dubbo-cluster 模块:** Dubbo 中负责管理集群的模块,提供了负载均衡、容错、路由等一系列集群相关的功能,最终的目的是将多个 Provider 伪装为一个 Provider,这样 Consumer 就可以像调用一个 Provider 那样调用 Provider 集群了.<br>

**dubbo-registry 模块:** Dubbo 中负责与多种开源注册中心进行交互的模块,提供注册中心的能力.其中, dubbo-registry-api 子模块是顶层抽象,其他子模块是针对具体开源注册中心组件的具体实现,例如,dubbo-registry-zookeeper 子模块是 Dubbo 接入 ZooKeeper 的具体实现.<br>

**dubbo-monitor 模块:** Dubbo 的监控模块,主要用于统计服务调用次数、调用时间以及实现调用链跟踪的服务.<br>

**dubbo-config 模块:** Dubbo 对外暴露的配置都是由该模块进行解析的.例如，dubbo-config-api 子模块负责处理 API 方式使用时的相关配置,dubbo-config-spring 子模块负责处理与 Spring 集成使用时的相关配置方式.有了 dubbo-config 模块,用户只需要了解 Dubbo 配置的规则即可,无须了解 Dubbo 内部的细节.<br>

**dubbo-metadata 模块:** Dubbo 的元数据模块. dubbo-metadata 模块的实现套路也是有一个 api 子模块进行抽象,然后其他子模块进行具体实现.<br>

**dubbo-configcenter 模块:** Dubbo 的动态配置模块,主要负责外部化配置以及服务治理规则的存储与通知,提供了多个子模块用来接入多种开源的外部配置中心组件.<br>



