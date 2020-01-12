---
layout:     post
title:      "springboot 启动流程"
date:       2019-11-08 17:00:00 +0800
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

查看run方法的实现并跟进底层,如下所示:
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

这里,首先通过SpringFactoriesLoader.loadFactoryNames(type, classLoader)方法,获取到META-INF/spring.factories文件中配置的ApplicationContextInitializer类名称;然后通过createSpringFactoriesInstances方法创建对应的Spring工厂实例;然后对Spring工厂实例排序(org.springframework.core.annotation.Order注解指定的顺序).

具体初始化的ApplicationContextInitializer实例包括以下几个:
![losnBj.png](https://s2.ax1x.com/2020/01/12/losnBj.png)
![losUb9.png](https://s2.ax1x.com/2020/01/12/losUb9.png)

ApplicationContextInitializer是Spring框架的类,这个类的主要目的就是在ConfigurableApplicationContext调用refresh()方法之前,回调这个类的initialize方法.通过  ConfigurableApplicationContext 的实例获取容器的环境Environment，从而实现对配置文件的修改完善等工作.

**这里的通过META-INF/spring.factories文件来加载类的方式,是SpringBoot中的一种SPI扩展机制.**

###		3.setListeners

初始化classpath下META-INF/spring.factories中已配置的ApplicationListener.  
ApplicationListener的加载过程和上面的ApplicationContextInitializer类的加载过程一样.  
ApplicationListener是spring的事件监听器,典型的观察者模式,通过ApplicationEvent类和ApplicationListener接口,可以实现对Spring容器全生命周期的监听.


###	容器初始化
容器初始化的流程在configureAndRefreshWebApplicationContext方法中实现.

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			wac.setId(idParam);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}

	wac.setServletContext(sc);
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
		wac.setConfigLocation(configLocationParam);
	}

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	}

	customizeContext(sc, wac);
	wac.refresh();
}
```

几个关键点如下:

#### 设置配置文件位置

```
String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}
```
配置文件的位置一般通过web.xml文件中的contextConfigLocation参数定义,例如:

```
<context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </context-param>
```

#### wac.refresh()

该方法实现spring容器初始化的完整逻辑,典型流程模板设计模式:

```
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

#####	ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

其方法内部调用refreshBeanFactory方法,完成两件事:
1.	创建一个DefaultListableBeanFactory实例,spring容器上下文XmlWebApplicationContext中会通过beanFactory属性来持有这个DefaultListableBeanFactory实例.
2.	将bean定义解析为BeanDefinition,存储到DefaultListableBeanFactory容器中.方法为loadBeanDefinitions.<br>
在该方法中,完成对配置文件中bean定义的解析,解析为BeanDefinition(它是spring对bean定义的抽象,封装bean的各种元数据信息,注意,它只是bean定义信息的封装,并不是bean实例).其流程大体为:  
	*	将配置文件的加载解析委托给XmlBeanDefinitionReader来处理.
	*	XmlBeanDefinitionReader委托给ResourceLoader来加载配置文件路径对应的文件加载为Resource(spring对配置配置文件的抽象)
	*	XmlBeanDefinitionReader将上述Resource资源列表委托给内部DefaultDocumentLoader,将资源处理为Document对象
	*	XmlBeanDefinitionReader将解析Document的工作委托给BeanDefinitionDocumentReader来处理.而BeanDefinitionDocumentReader内部会将解析工作再委托给BeanDefinitionParserDelegate来执行.在BeanDefinitionParserDelegate的解析工作中,会将配置文件中的各标签区分为两类来进行解析:
		*	默认命名空间元素(这里会针对import,alias,bean,beans来做相应处理)
		*	自定义命名空间元素,这里会根据标签来找到对应的NamespaceHandler,完成处理(例如,我们常见的\<context:component-scan base-package="com.simba.abc"/>,就是通过其对应的ContextNamespaceHandler来处理的,在这个handler中,会注册component-scan对应的BeanDefinitionParser,通过对应BeanDefinitionParser#parse方法来处理完成注解bean的扫描并解析为BeanDefinition.更进一步的内部逻辑这里就不讲了.翻源码吧..)

<font color="red">至此,解析为BeanDefinition的工作完成.总结一句就是,通过各种委托来处理解析工作.</font>



#####	prepareBeanFactory(beanFactory);

完成对BeanFactory的一些特性设置工作.具体的,翻源码看注释吧,写的很清楚



#####	postProcessBeanFactory(beanFactory);

允许在上下文子类中对bean工厂进行后处理。  

标准化初始化后,修改应用程序上下文的内部bean工厂.在那时候,所有bean定义都将已经被加载完成,但还未实例化任何bean.这允许在某些ApplicationContext实现中注册特殊的BeanPostProcessor等.

因为我们这里的ApplicationContext实现类为XmlWebApplicationContext,它postProcessBeanFactory方法继承自其父类AbstractRefreshableWebApplicationContext,注册了一个与Servlet一些设置相关的后置处理器ServletContextAwareProcessor



#####	