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


##	1.	获取并启动监听器

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


##	2.	构造应用上下文环境

应用上下文环境包括计算机环境,Java环境,Spring运行环境,Spring项目的配置(SpringBoot中就是那个熟悉的application.properties/yml)等等.

###	2.1	prepareEnvironment方法

```
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
	// Create and configure the environment
	//创建并配置相应的环境
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	configureEnvironment(environment, applicationArguments.getSourceArgs());

	//根据用户配置，配置 environment系统环境
	ConfigurationPropertySources.attach(environment);

	// 发布ApplicationEnvironmentPreparedEvent事件
	// 启动相应的监听器,其中一个重要的监听器ConfigFileApplicationListener就是加载项目配置文件的监听器
	listeners.environmentPrepared(environment);
	bindToSpringApplication(environment);
	if (!this.isCustomEnvironment) {
		environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
				deduceEnvironmentClass());
	}
	ConfigurationPropertySources.attach(environment);
	return environment;
}
```

方法中主要完成的工作，首先是创建并按照相应的应用类型配置相应的环境，然后根据用户的配置，配置系统环境，然后启动监听器，并加载系统配置文件

####	2.1.1	getOrCreateEnvironment

看一下getOrCreateEnvironment.

```
private ConfigurableEnvironment getOrCreateEnvironment() {
	if (this.environment != null) {
		return this.environment;
	}
	switch (this.webApplicationType) {
	case SERVLET:
		return new StandardServletEnvironment();
	case REACTIVE:
		return new StandardReactiveWebEnvironment();
	default:
		return new StandardEnvironment();
	}
}
```

方法中完成的工作就是,根据不同的应用类型初始化不同的系统环境实例.

####	2.1.2	configureEnvironment

```
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
	if (this.addConversionService) {
		ConversionService conversionService = ApplicationConversionService.getSharedInstance();
		environment.setConversionService((ConfigurableConversionService) conversionService);
	}
	// 将main 函数的args封装成 SimpleCommandLinePropertySource 加入环境中.
	configurePropertySources(environment, args);

	// 激活相应的配置文件
	configureProfiles(environment, args);
}
```

在configurePropertySources(environment,args);中将args封装成了SimpleCommandLinePropertySource并加入到了environment中.  
configureProfiles(environment, args);根据启动参数激活了相应的配置文件.  
<font color="red">spring profile多环境配置文件激活就是在这里.</font>


####	2.1.3	listeners.environmentPrepared

发布ApplicationEnvironmentPreparedEvent事件.  
这里有一个重要的监听器ConfigFileApplicationListener.  
看下这个类的注释:
```
/**
 * {@link EnvironmentPostProcessor} that configures the context environment by loading
 * properties from well known file locations. By default properties will be loaded from
 * 'application.properties' and/or 'application.yml' files in the following locations:
 * <ul>
 * <li>file:./config/</li>
 * <li>file:./</li>
 * <li>classpath:config/</li>
 * <li>classpath:</li>
 * </ul>
 * The list is ordered by precedence (properties defined in locations higher in the list
 * override those defined in lower locations).
 * <p>
 * Alternative search locations and names can be specified using
 * {@link #setSearchLocations(String)} and {@link #setSearchNames(String)}.
 * <p>
 * Additional files will also be loaded based on active profiles. For example if a 'web'
 * profile is active 'application-web.properties' and 'application-web.yml' will be
 * considered.
 * <p>
 * The 'spring.config.name' property can be used to specify an alternative name to load
 * and the 'spring.config.location' property can be used to specify alternative search
 * locations or specific files.
 * <p>
 *
 * @author Dave Syer
 * @author Phillip Webb
 * @author Stephane Nicoll
 * @author Andy Wilkinson
 * @author Eddú Meléndez
 * @author Madhura Bhave
 * @since 1.0.0
 */
public class ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {
	...
}
```

这个监听器默认的从注释中<ul>标签所示的几个位置加载配置文件,并将其加入上下文的environment变量中.


##	3.	初始化应用上下文

```
protected ConfigurableApplicationContext createApplicationContext() {
	Class<?> contextClass = this.applicationContextClass;
	if (contextClass == null) {
		try {
			switch (this.webApplicationType) {
			case SERVLET:
				contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
				break;
			case REACTIVE:
				contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
				break;
			default:
				contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
			}
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalStateException(
					"Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
		}
	}
	return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

对应三种应用类型,以web工程为例,其上下文为AnnotationConfigServletWebServerApplicationContext,来看一下它的继承体系:
![lHCN5D.png](https://s2.ax1x.com/2020/01/13/lHCN5D.png)

应用上下文对IoC容器是持有的关系,他的一个属性beanFactory就是IoC容器(DefaultListableBeanFactory).所以他们之间是持有,和扩展的关系.

AnnotationConfigServletWebServerApplicationContext继承体系中有父类GenericApplicationContext,该类的构造方法中会初始化beanFactory属性为DefaultListableBeanFactory
```
/**
 * Create a new GenericApplicationContext.
 * @see #registerBeanDefinition
 * @see #refresh
 */
public GenericApplicationContext() {
	this.beanFactory = new DefaultListableBeanFactory();
}
```

``context就是我们熟悉的上下文(也有人称之为容器,都可以,看个人爱好和理解),而beanFactory就是我们所说的IoC容器的真实面孔了.``