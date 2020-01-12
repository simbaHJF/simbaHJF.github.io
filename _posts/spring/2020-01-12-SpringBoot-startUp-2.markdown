---
layout:     post
title:      "springboot启动流程----SpringApplication的run方法"
date:       2020-01-12 22:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---

#	SpringApplication实例的run方法

方法源码如下:
```
/**
 * Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {
	//记录程序运行时间
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();

	//Spring的上下文:ConfigurableApplicationContext
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	configureHeadlessProperty();

	//从META-INF/spring.factories中获取监听器
	SpringApplicationRunListeners listeners = getRunListeners(args);
	//启动监听器,发布ApplicationStartingEvent事件
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

		//构造应用上下文环境
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

		//处理需要忽略的bean
		configureIgnoreBeanInfo(environment);

		//打印banner
		Banner printedBanner = printBanner(environment);

		//初始化应用上下文
		context = createApplicationContext();

		//实例化SpringBootExceptionReporter.class,用来支持报告关于启动的错误
		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context);

		//刷新应用上下文前的准备阶段
		prepareContext(context, environment, listeners, applicationArguments, printedBanner);

		//刷新应用上下文
		refreshContext(context);

		//刷新应用上下文后的扩展接口
		afterRefresh(context, applicationArguments);

		//时间记录停止
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
		}

		//发布容器启动完成事件
		listeners.started(context);
		callRunners(context, applicationArguments);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```

下面进行具体分析


##	获取并启动监听器

```
private SpringApplicationRunListeners getRunListeners(String[] args) {
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
	return new SpringApplicationRunListeners(logger,
			getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

从META-INF/spring.factories获取指定的SpringApplicationRunListener实例  
具体包括:EventPublishingRunListener.如下:
![lTrZVA.png](https://s2.ax1x.com/2020/01/12/lTrZVA.png)
EventPublishingRunListener监听器是Spring容器的启动监听器.调用其starting方法,将发布ApplicationStartingEvent事件.

``
这里稍微总结一下SpringApplicationRunListener和ApplicationListener.  
ApplicationListener是在创建SpringApplication实例的时候,根据META-INF/spring.factories文件初始化的的监听器,实现了响应事件发生后具体的执行逻辑.  
而SpringApplicationRunListener是启动监听器,是用来触发上面那些具体监听器的,触发方式就是通过发布相应的事件,也就是SpringApplicationEvent.
``