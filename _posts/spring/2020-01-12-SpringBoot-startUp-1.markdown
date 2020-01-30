---
layout:     post
title:      "springboot启动流程----SpringApplication类初始化"
date:       2020-01-12 20:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---

#	SpringApplication类初始化过程

##	SpringBoot项目的main方法入口

```
@SpringBootApplication
public class SpringBootStudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootStudyApplication.class, args);
    }

}
```

查看SpringApplication.run方法的实现并跟进底层,如下所示:
```
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
	return new SpringApplication(primarySources).run(args);
}
```

上面过程是,首先创建一个SpringApplication实例,然后调用SpringApplication实例的run方法.下面依次来进行分析.


##	SpringApplication实例的创建

跟进SpringApplication的构造方法到底层,代码如下:
```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	//推断应用类型，后面会根据类型初始化对应的环境。常用的一般都是servlet环境
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	//初始化classpath下 META-INF/spring.factories中已配置的ApplicationContextInitializer
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	//初始化classpath下所有已配置的 ApplicationListener
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	//根据调用栈，推断出 main 方法的类名
	this.mainApplicationClass = deduceMainApplicationClass();
}
```
下面来依次分析下上述方法内部逻辑:

###		1.WebApplicationType.deduceFromClasspath()

```
/**
 * The application should not run as a web application and should not start an
 * embedded web server.
 */
NONE,
/**
 * The application should run as a servlet-based web application and should start an
 * embedded servlet web server.
 */
SERVLET,
/**
 * The application should run as a reactive web application and should start an
 * embedded reactive web server.
 */
REACTIVE;
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
		"org.springframework.web.context.ConfigurableWebApplicationContext" };
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";
private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";
private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

/**
 * 判断 应用的类型
 * NONE: 应用程序不是web应用，也不应该用web服务器去启动
 * SERVLET: 应用程序应作为基于servlet的web应用程序运行，并应启动嵌入式servlet web（tomcat）服务器。
 * REACTIVE: 应用程序应作为 reactive web应用程序运行，并应启动嵌入式 reactive web服务器。
 * @return
 */
static WebApplicationType deduceFromClasspath() {
	if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
			&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
		return WebApplicationType.REACTIVE;
	}
	for (String className : SERVLET_INDICATOR_CLASSES) {
		if (!ClassUtils.isPresent(className, null)) {
			return WebApplicationType.NONE;
		}
	}
	return WebApplicationType.SERVLET;
}
```

返回类型是WebApplicationType的枚举值,WebApplicationType有三个枚举,三个枚举的意义如注释  
具体判断逻辑如下  
*	WebApplicationType.REACTIVE  classpath下存在org.springframework.web.reactive.DispatcherHandler
*	WebApplicationType.SERVLET classpath下存在javax.servlet.Servlet或者org.springframework.web.context.ConfigurableWebApplicationContext
*	WebApplicationType.NONE 不满足以上条件

###		2.setInitializers

首先通过getSpringFactoriesInstances(ApplicationContextInitializer.class)方法获取ApplicationContextInitializer类型的集合  
其原理是,初始化classpath下 META-INF/spring.factories中已配置的ApplicationContextInitializer.
```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
	ClassLoader classLoader = getClassLoader();
	// Use names and ensure unique to protect against duplicates
	Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}
```

这里,首先通过SpringFactoriesLoader.loadFactoryNames(type, classLoader)方法,获取到META-INF/spring.factories文件中配置的ApplicationContextInitializer类名称;然后通过createSpringFactoriesInstances方法创建对应的ApplicationContextInitializer实例;然后对它们排序(org.springframework.core.annotation.Order注解指定的顺序).

具体初始化的ApplicationContextInitializer实例包括以下几个:
![losnBj.png](https://s2.ax1x.com/2020/01/12/losnBj.png)
![losUb9.png](https://s2.ax1x.com/2020/01/12/losUb9.png)

ApplicationContextInitializer是Spring框架的类,这个类的主要目的就是在ConfigurableApplicationContext调用refresh()方法之前,回调这个类的initialize方法.通过  ConfigurableApplicationContext 的实例获取容器的环境Environment，从而实现对配置文件的修改完善等工作.

**这里的通过META-INF/spring.factories文件来加载类的方式,是SpringBoot中的一种SPI扩展机制.**

###		3.setListeners

初始化classpath下META-INF/spring.factories中已配置的ApplicationListener.  
ApplicationListener的加载过程和上面的ApplicationContextInitializer类的加载过程一样.  
ApplicationListener是spring的事件监听器,典型的观察者模式,通过ApplicationEvent类和ApplicationListener接口,可以实现对Spring容器全生命周期的监听.  
具体初始化的ApplicationListener实例包括如下:
![lTrZVA.png](https://s2.ax1x.com/2020/01/12/lTrZVA.png)
![lTcKoj.png](https://s2.ax1x.com/2020/01/12/lTcKoj.png)






#####	prepareBeanFactory(beanFactory);

完成对BeanFactory的一些特性设置工作.具体的,翻源码看注释吧,写的很清楚



#####	postProcessBeanFactory(beanFactory);

允许在上下文子类中对bean工厂进行后处理。  

标准化初始化后,修改应用程序上下文的内部bean工厂.在那时候,所有bean定义都将已经被加载完成,但还未实例化任何bean.这允许在某些ApplicationContext实现中注册特殊的BeanPostProcessor等.

因为我们这里的ApplicationContext实现类为XmlWebApplicationContext,它postProcessBeanFactory方法继承自其父类AbstractRefreshableWebApplicationContext,注册了一个与Servlet一些设置相关的后置处理器ServletContextAwareProcessor



#####	