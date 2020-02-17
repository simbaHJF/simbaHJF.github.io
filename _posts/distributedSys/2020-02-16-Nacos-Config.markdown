---
layout:     post
title:      "Nacos配置中心原理"
date:       2020-02-17 00:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 微服务

---


> <font color="red"> nacos版本:	1.1.4 </font>


本文Nacos配置中心客户端原理进行分析,以nacos官方文档,java SDK的方式为基础,至于与springboot继承的注解引入方式,会在后面文章中分析

##	一.	监听配置

先看下官方文档给出的示例代码:

```
String serverAddr = "{serverAddr}";
String dataId = "{dataId}";
String group = "{group}";
Properties properties = new Properties();
properties.put("serverAddr", serverAddr);
ConfigService configService = NacosFactory.createConfigService(properties);
String content = configService.getConfig(dataId, group, 5000);
System.out.println(content);
configService.addListener(dataId, group, new Listener() {
	@Override
	public void receiveConfigInfo(String configInfo) {
		System.out.println("recieve1:" + configInfo);
	}
	@Override
	public Executor getExecutor() {
		return null;
	}
});

// 测试让主线程不退出，因为订阅配置是守护线程，主线程退出守护线程就会退出。 正式代码中无需下面代码
while (true) {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

这里主要分三步:
*	创建ConfigService对象
*	获取配置
*	添加监听器

下面来具体分析


###		一.1		&emsp;&emsp;	NacosFactory.createConfigService(properties);

来看下NacosFactory.createConfigService(properties);方法的内部实现

```
public static ConfigService createConfigService(Properties properties) throws NacosException {
    return ConfigFactory.createConfigService(properties);
}
```

继续跟进
```
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable e) {
        throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
    }
}
```

这里我们看到,以反射的方式创建了一个NacosConfigService实例,再来看下NacosConfigService的构造方法:

```
public NacosConfigService(Properties properties) throws NacosException {
    String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
    if (StringUtils.isBlank(encodeTmp)) {
        encode = Constants.ENCODE;
    } else {
        encode = encodeTmp.trim();
    }
    initNamespace(properties);
    agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
    agent.start();
    worker = new ClientWorker(agent, configFilterChainManager, properties);
}
```
这里完成如下几件事:
*	进行编码设置
*	初始化命名空间(如果没有设置的话,这块将会是空字符串)
*	设置http代理,也就是agent;并启动agent
*	创建ClientWorker(Longpolling,长轮询)

前两步很简单,没什么好说的,这里主要说下后面两步.


####	一.1.1  	&emsp;&emsp;  设置http代理,启动agent

关注几个类HttpAgent,MetricsHttpAgent,ServerHttpAgent.

HttpAgent是一个接口,抽象了各个http请求方法,是nacos config的client中一个http代理的顶层抽象:
```
public interface HttpAgent {
    /**
     * start to get nacos ip list
     */
    void start() throws NacosException;

    /**
     * invoke http get method
     */
    HttpResult httpGet(String path, List<String> headers, List<String> paramValues, String encoding, long readTimeoutMs) throws IOException;

    /**
     * invoke http post method
     */
    HttpResult httpPost(String path, List<String> headers, List<String> paramValues, String encoding, long readTimeoutMs) throws IOException;

    /**
     * invoke http delete method
     */
    HttpResult httpDelete(String path, List<String> headers, List<String> paramValues, String encoding, long readTimeoutMs) throws IOException;

    /**
     * get name
     */
    String getName();

    /**
     * get namespace
     */
    String getNamespace();

    /**
     * get tenant
     */
    String getTenant();

    /**
     * get encode
     */
    String getEncode();
}
```






###		一.2  &emsp;&emsp;  configService.getConfig(dataId, group, 5000);


###		一.3  &emsp;&emsp;  configService.addListener