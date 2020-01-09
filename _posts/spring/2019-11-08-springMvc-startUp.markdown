---
layout:     post
title:      "springmvc 启动流程"
date:       2019-11-08 17:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---

##	servlet容器启动

在web.xml文件中一般会配置servlet容器上下文监听器,如下:
```
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

当tomcat容器启动完成后,会执行ContextLoaderListener#contextInitialized这一回调方法.来完成spring中容器的初始化.

##	spring容器的启动
```
public void contextInitialized(ServletContextEvent event) {
	initWebApplicationContext(event.getServletContext());
}
```
进入initWebApplicationContext方法内部,具体执行逻辑如下:
```
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
				"Cannot initialize context because there is already a root application context present - " +
				"check whether you have multiple ContextLoader* definitions in your web.xml!");
	}
	servletContext.log("Initializing Spring root WebApplicationContext");
	Log logger = LogFactory.getLog(ContextLoader.class);
	if (logger.isInfoEnabled()) {
		logger.info("Root WebApplicationContext: initialization started");
	}
	long startTime = System.currentTimeMillis();
	try {
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent ->
					// determine parent for root web application context, if any.
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

		ClassLoader ccl = Thread.currentThread().getContextClassLoader();
		if (ccl == ContextLoader.class.getClassLoader()) {
			currentContext = this.context;
		}
		else if (ccl != null) {
			currentContextPerThread.put(ccl, this.context);
		}

		if (logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
		}

		return this.context;
	}
	catch (RuntimeException | Error ex) {
		logger.error("Context initialization failed", ex);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}
}
```

###	容器的创建
首先来看一下spring是如何决定创建的容器类型,其逻辑如下:  
this.context = createWebApplicationContext(servletContext);

跟进方法内部,具体逻辑为:
```
protected Class<?> determineContextClass(ServletContext servletContext) {
	String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
	if (contextClassName != null) {
		try {
			return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
		}
		catch (ClassNotFoundException ex) {
			throw new ApplicationContextException(
					"Failed to load custom context class [" + contextClassName + "]", ex);
		}
	}
	else {
		contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
		try {
			return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
		}
		catch (ClassNotFoundException ex) {
			throw new ApplicationContextException(
					"Failed to load default context class [" + contextClassName + "]", ex);
		}
	}
}
```

它的过程如下:
*	如果配置了contextClass参数,则容器类型即为该参数值指定的class类型.例如:

```
<context-param>
	<param-name>contextClass</param-name>
	<param-value>
		org.springframework.web.context.support.AnnotationConfigWebApplicationContext
	</param-value>
</context-param>
```

*	否则执行  
contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());  
来获取容器类型.其中defaultStrategies会加载ContextLoader.properties文件中的属性,在该文件中,对容器class类型进行了默认设置.<br>
ContextLoader.properties内容如下:

```
# Default WebApplicationContext implementation class for ContextLoader.
# Used as fallback when no explicit context implementation has been specified as context-param.
# Not meant to be customized by application developers.

org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```

*	通过反射,创建对应容器class类型的一个实例.

下面来看下XmlWebApplicationContext的类图:
![Qv3iM6.png](https://s2.ax1x.com/2019/12/21/Qv3iM6.png)


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