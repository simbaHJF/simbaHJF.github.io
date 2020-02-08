---
layout:     post
title:      "springboot启动流程----IOC容器的初始化过程----BeanDefinition解析"
date:       2020-01-22 20:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---


#	AbstractApplicationContext的refresh()

该方法是容器初始化的关键方法,采用典型的模板方法设计模式.  
先来看先源码:

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


##	1.	ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	return getBeanFactory();
}
```

上述源码中调用的两个方法,在GenericApplicationContext中都进行了重写,跟进源码中,发现其实现很简单,没有做什么复杂处理了.这点跟原来只使用SpringMVC时,执行AbstractRefreshableApplicationContext类的refreshBeanFactory方法有很大不同,原来的逻辑中,会在这个方法里完成配置文件的BeanDefinition的解析和注册.而现在在GenericApplicationContext中就很简单了,只是几个简单的属性设置.


##	2.	prepareBeanFactory(beanFactory);

完成对BeanFactory的一些特性设置工作.
```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    // 配置类加载器：默认使用当前上下文的类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    // 配置EL表达式：在Bean初始化完成，填充属性的时候会用到
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 添加属性编辑器 PropertyEditor
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 添加Bean的后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 忽略装配以下指定的类
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 将以下类注册到 beanFactory（DefaultListableBeanFactory） 的resolvableDependencies属性中
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    // 将早期后处理器注册为application监听器，用于检测内部bean
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    //如果当前BeanFactory包含loadTimeWeaver Bean，说明存在类加载期织入AspectJ，
    // 则把当前BeanFactory交给类加载期BeanPostProcessor实现类LoadTimeWeaverAwareProcessor来处理，
    // 从而实现类加载期织入AspectJ的目的。
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    // 将当前环境变量（environment） 注册为单例bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    // 将当前系统配置（systemProperties） 注册为单例Bean
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    // 将当前系统环境 （systemEnvironment） 注册为单例Bean
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

##	3.	postProcessBeanFactory(beanFactory);

允许在上下文子类中对beanFactory进行后处理.  
标准化初始化后,修改应用程序上下文的内部bean工厂.在那时候,所有bean定义都将已经被加载完成,但还未实例化任何bean.简言之,BeanFactory的后置处理器可以修改BeanDefinition的属性信息.

##	4.	invokeBeanFactoryPostProcessors(beanFactory);(重点方法)

IoC容器的BeanDefinition解析注册过程包括三个步骤,在invokeBeanFactoryPostProcessors()方法中完成了这三个步骤.先来对这三个步骤进行一下总结

###	4.1	第一步:Resource定位

在SpringBoot中,我们都知道他的包扫描是从主类所在的包开始扫描的,refresh()方法的prepareContext()方法中,会先将主类解析成BeanDefinition,然后在refresh()方法的invokeBeanFactoryPostProcessors()方法中解析主类的BeanDefinition获取basePackage的路径.这样就完成了定位的过程.其次SpringBoot的各种starter是通过SPI扩展机制实现的自动装配,SpringBoot的自动装配同样也是在invokeBeanFactoryPostProcessors()方法中实现的.还有一种情况,在SpringBoot中有很多的@EnableXXX注解,细心点进去看的应该就知道其底层是@Import注解,在invokeBeanFactoryPostProcessors()方法中也实现了对该注解指定的配置类的定位加载.  
prepareContext()方法在上一篇文章中讲过.

常规的在SpringBoot中有三种实现定位,第一个是主类所在包的,第二个是SPI扩展机制实现的自动装配（比如各种starter）,第三种就是@Import注解指定的类.(对于非常规的不说了)

###	4.2	第二步:BeanDefinition的解析载入

在Resource定位后,紧接着就是BeanDefinition的解析载入.所谓解析载入就是通过上面定位得到的basePackage,SpringBoot会将该路径拼接成:classpath\*:org/springframework/boot/demo/\*\*/\*.class这样的形式,然后一个叫做PathMatchingResourcePatternResolver的类会将该路径下所有的.class文件都加载进来,然后遍历判断是不是有@Component注解,如果有的话,就是需要装载的BeanDefinition.载入的大致过程即是如此.

``
TIPS:  
@Configuration,@Controller,@Service等注解底层都是@Component注解,只不过包装了一层罢了.
``

###	4.3	第三步:注册BeanDefinition

这个过程通过调用BeanDefinitionRegister接口的实现来完成.这个注册过程把载入过程中解析得到的BeanDefinition向IoC容器进行注册.在IoC容器中将BeanDefinition注入到一个ConcurrentHashMap中,IoC容器就是通过这个HashMap来持有这些BeanDefinition数据的.比如DefaultListableBeanFactory中的beanDefinitionMap属性.


###	4.4	BeanDefinition解析载入和注册的源码过程详述

BeanDefinition解析载入和注册过程主要在AbstractApplicationContext的refresh方法中invokeBeanFactoryPostProcessors(beanFactory)方法逻辑中实现.

a.
```
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```

b.  
跟进PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());方法

```
public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
	// Invoke BeanDefinitionRegistryPostProcessors first, if any.
	Set<String> processedBeans = new HashSet<>();
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				registryProcessors.add(registryProcessor);
			}
			else {
				regularPostProcessors.add(postProcessor);
			}
		}
		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// Separate between BeanDefinitionRegistryPostProcessors that implement
		// PriorityOrdered, Ordered, and the rest.
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
		// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();
		// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();
		// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
		boolean reiterate = true;
		while (reiterate) {
			reiterate = false;
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}
		// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}
	else {
		// Invoke factory processors registered with the context instance.
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}
	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let the bean factory post-processors apply to them!
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
	// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (processedBeans.contains(ppName)) {
			// skip - already processed in first phase above
		}
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}
	// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);
	// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
	// Finally, invoke all other BeanFactoryPostProcessors.
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}
```

这里重点是invokeBeanDefinitionRegistryPostProcessors方法的执行,可以看到它有多处执行过程.简单总结一下就是:

这里会获取BeanDefinitionRegistryPostProcessor,然后按照一定的要求和先后顺序,一部分一部分的将其添加到currentRegistryProcessors中来执行.  
当前一部分BeanDefinitionRegistryPostProcessor执行完成后,清空currentRegistryProcessors,然后再获取下一部分BeanDefinitionRegistryPostProcessor来执行处理.  
因此BeanDefinitionRegistryPostProcessor的执行是有优先级先后顺序的.

**<font color="red">这里最重要的一个BeanDefinitionRegistryPostProcessor是ConfigurationClassPostProcessor,它实现了PriorityOrdered,是优先执行的一个BeanDefinitionRegistryPostProcessor.ConfigurationClassPostProcessor中完成了BeanDefinition的扫描解析和注册工作.</font>**

c.
```
private static void invokeBeanDefinitionRegistryPostProcessors(
		Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {
	for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanDefinitionRegistry(registry);
	}
}
```

d.
由上面c会进入到ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法

```
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
	int registryId = System.identityHashCode(registry);
	if (this.registriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(
				"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
	}
	if (this.factoriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(
				"postProcessBeanFactory already called on this post-processor against " + registry);
	}
	this.registriesPostProcessed.add(registryId);
	processConfigBeanDefinitions(registry);
}
```

e.

```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	String[] candidateNames = registry.getBeanDefinitionNames();
	for (String beanName : candidateNames) {
		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
		if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
		}
		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
	}
	// Return immediately if no @Configuration classes were found
	if (configCandidates.isEmpty()) {
		return;
	}
	// Sort by previously determined @Order value, if applicable
	configCandidates.sort((bd1, bd2) -> {
		int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
		int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
		return Integer.compare(i1, i2);
	});
	// Detect any custom bean name generation strategy supplied through the enclosing application context
	SingletonBeanRegistry sbr = null;
	if (registry instanceof SingletonBeanRegistry) {
		sbr = (SingletonBeanRegistry) registry;
		if (!this.localBeanNameGeneratorSet) {
			BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
			if (generator != null) {
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}
	}
	if (this.environment == null) {
		this.environment = new StandardEnvironment();
	}
	// Parse each @Configuration class
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);
	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
	do {
		parser.parse(candidates);
		parser.validate();
		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
		configClasses.removeAll(alreadyParsed);
		// Read the model and create bean definitions based on its content
		if (this.reader == null) {
			this.reader = new ConfigurationClassBeanDefinitionReader(
					registry, this.sourceExtractor, this.resourceLoader, this.environment,
					this.importBeanNameGenerator, parser.getImportRegistry());
		}
		this.reader.loadBeanDefinitions(configClasses);
		alreadyParsed.addAll(configClasses);
		candidates.clear();
		if (registry.getBeanDefinitionCount() > candidateNames.length) {
			String[] newCandidateNames = registry.getBeanDefinitionNames();
			Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
			Set<String> alreadyParsedClasses = new HashSet<>();
			for (ConfigurationClass configurationClass : alreadyParsed) {
				alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
			}
			for (String candidateName : newCandidateNames) {
				if (!oldCandidateNames.contains(candidateName)) {
					BeanDefinition bd = registry.getBeanDefinition(candidateName);
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
							!alreadyParsedClasses.contains(bd.getBeanClassName())) {
						candidates.add(new BeanDefinitionHolder(bd, candidateName));
					}
				}
			}
			candidateNames = newCandidateNames;
		}
	}
	while (!candidates.isEmpty());
	// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
	if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
		sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
	}
	if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
		// Clear cache in externally provided MetadataReaderFactory; this is a no-op
		// for a shared cache since it'll be cleared by the ApplicationContext.
		((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
	}
}
```

这里debug进来,发现candidates中只有一个类,就是我们的主类:
![1VCfne.png](https://s2.ax1x.com/2020/01/23/1VCfne.png)

跟进do{}whie()循环

f.

```
public void parse(Set<BeanDefinitionHolder> configCandidates) {
	for (BeanDefinitionHolder holder : configCandidates) {
		BeanDefinition bd = holder.getBeanDefinition();
		try {
			if (bd instanceof AnnotatedBeanDefinition) {
				parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
			}
			else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
				parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
			}
			else {
				parse(bd.getBeanClassName(), holder.getBeanName());
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
		}
	}
	this.deferredImportSelectorHandler.process();
}
```

debug进来,会进入到第一个条件判断分支,因为主类的@SpringBootApplication注解其实内部引入了@Component注解
![1VPeE9.png](https://s2.ax1x.com/2020/01/23/1VPeE9.png)

然后跟进parse方法.
```
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}
	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	if (existingClass != null) {
		if (configClass.isImported()) {
			if (existingClass.isImported()) {
				existingClass.mergeImportedBy(configClass);
			}
			// Otherwise ignore new imported config class; existing non-imported class overrides it.
			return;
		}
		else {
			// Explicit bean definition found, probably replacing an import.
			// Let's remove the old one and go with the new one.
			this.configurationClasses.remove(configClass);
			this.knownSuperclasses.values().removeIf(configClass::equals);
		}
	}
	// Recursively process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass);
	do {
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
	}
	while (sourceClass != null);
	this.configurationClasses.put(configClass, configClass);
}
```

这里doProcessConfigurationClass是关键


g.

```
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
		throws IOException {
	if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
		// Recursively process any member (nested) classes first
		//首先递归处理内部类,(SpringBoot项目的主类一般没有内部类)
		processMemberClasses(configClass, sourceClass);
	}
	// Process any @PropertySource annotations
	// 针对 @PropertySource 注解的属性配置处理
	// a.
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
		else {
			logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}
	// Process any @ComponentScan annotations
	// 根据 @ComponentScan 注解,扫描项目中的Bean(SpringBoot 启动类上有该注解)
	// b.
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			// 配置类被标注了@ComponentScan注解 -> 立即执行扫描（SpringBoot项目是如何开始包扫描的,这就是关键了）
			// c.
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			// 检查被扫描到的BeanDefinition中是否有更进一步的配置类,如果需要的话递归的执行parse
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
				if (bdCand == null) {
					bdCand = holder.getBeanDefinition();
				}
				// 检查是否是ConfigurationClass(是否有configuration/component两个注解),如果是,递归查找该类相关联的配置类.
                // 所谓相关的配置类,比如@Configuration中的@Bean定义的bean.或者在有@Component注解的类上继续存在@Import注解.
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
					parse(bdCand.getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}
	// Process any @Import annotations
	// 递归处理 @Import 注解(SpringBoot项目中经常用的各种@Enable*** 注解基本都是封装的@Import)
	// d.
	processImports(configClass, sourceClass, getImports(sourceClass), true);
	// Process any @ImportResource annotations
	// 处理 @ImportResource注解
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}
	// Process individual @Bean methods
	// 处理独立的@Bean注解方法
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}
	// Process default methods on interfaces
	processInterfaces(configClass, sourceClass);
	// Process superclass, if any
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}
	// No superclass -> processing is complete
	return null;
}
```

这里有几个点:

*	a处代码:  
	获取主类上的@PropertySource注解,解析该注解并将该注解指定的properties配置文件中的值存储到Spring的Environment中,Environment接口提供方法区读取配置文件中的值,参数是properties文件中定义的key值.
*	b处代码:  
	解析主类上的@ComponentScan注解.
*	c处代码:  
	解析扫描解析BeanDefinition
*	d处代码:  
	解析主类上的@Import注解,并加载该注解指定的配置类


这里再重点说一下c处代码逻辑

```
// ComponentScanAnnotationParser类
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
            componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
    ...
    // 根据 declaringClass （如果是SpringBoot项目，则参数为主类的全路径名）
    if (basePackages.isEmpty()) {
        basePackages.add(ClassUtils.getPackageName(declaringClass));
    }
    ...
    // 根据basePackages扫描类,并完成BeanDefinition的解析和注册
    return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

debug,跟进scanner.doScan方法,会来到如下代码段:
![1Q8Zzd.png](https://s2.ax1x.com/2020/01/29/1Q8Zzd.png)


<font color="red">
	这里总结一下:  
	<br>
	1.	由上面debug显示,packageSearchPath = "classpath*:com/simba/springbootstudy/**/*.class",所以扫描的路径是匹配前面字符串正则的路径.
	<br>
	2.	如果主类上配置了@ComponentScan注解,那么就不会按主类所在包路径去生成扫描路径的正则字符串,因此这里如果不注意的话会造成一切bean扫描不到的报错情况,需要注意.这一点看后面的举例说明.
</font>


这里说明一下上面的第2点.比如下面的包结构加上@ComponentScan注解:
![1QGyAf.png](https://s2.ax1x.com/2020/01/29/1QGyAf.png)

来看下这种情况下的扫描路径:
![1QGTEV.png](https://s2.ax1x.com/2020/01/29/1QGTEV.png)

因此,这个时候和主类在同一个包路径下的TestBean就无法被扫描到.一定要注意!!!!!!  


剩下的,在c中的代码逻辑中完成的功能就是通过ClassPathBeanDefinitionScanner的doScan方法扫描解析BeanDefinition和注册了,这里就不在详述了.


BeanDefinition的解析注册工作就分析到这里,下一篇分析在BeanDefinition基础上,对bean进行创建和初始化.
