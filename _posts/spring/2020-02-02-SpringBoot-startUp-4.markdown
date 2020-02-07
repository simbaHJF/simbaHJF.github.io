---
layout:     post
title:      "springboot启动流程----IOC容器的初始化过程----依赖注入"
date:       2020-02-02 20:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---


#	AbstractApplicationContext的refresh()

同样,还是先放上这个顶层模板方法的代码:

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


##	一.	registerBeanPostProcessors(beanFactory);

注册一些内部BeanPostProcessor.将这些内部BeanPostProcessor注册为Bean,这些BeanPostProcessor会在bean创建过程中进行前后置方法拦截调用以完成一些功能.  

其中最重要也是最常见的两个BeanPostProcessor即是:

*	AutowiredAnnotationBeanPostProcessor,   对@Autowired注解进行处理的(当然还包括一些其他注解的处理)
*	CommonAnnotationBeanPostProcessor,	对@Resource注解进行处理(当然还包括一些其他注解的处理)

他们的处理逻辑类似,主要就是将Bean中符合条件(需要进行依赖注入)的属性获取到,然后将这些信息合并到BeanDefinition中,以便于后面进行依赖注入.



##	二.	finishBeanFactoryInitialization(beanFactory);

```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}
	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}
	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}
	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);
	// Allow for caching all bean definition metadata, not expecting further changes.
	beanFactory.freezeConfiguration();
	// Instantiate all remaining (non-lazy-init) singletons.
	beanFactory.preInstantiateSingletons();
}
```

这里,bean的创建和初始化(也就是整个依赖注入)在代码beanFactory.preInstantiateSingletons()中实现,来看一下:

```
public void preInstantiateSingletons() throws BeansException {
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}
	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}
	// Trigger post-initialization callback for all applicable beans...
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```


这里getBean方法就是依赖注入的核心重点方法,本文后面就是重点分析该方法.


##	三.	getBean方法

跟进getBean方法调用底层,内部代码逻辑如下

```
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
		@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	final String beanName = transformedBeanName(name);
	Object bean;

	// 1.检查缓存中或者实例工厂中是否有对应的实例
    // 因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖
    // Spring在创建bean的时候不会等bean创建完成就会将bean的ObjectFactory提早曝光
    // 也就是将ObjectFactory加入到缓存中，一旦下一个要创建的bean需要依赖上个bean则直接使用ObjectFactory
    // 2.spring 默认是单例的，如果能获取到直接返回，提高效率。
	// Eagerly check singleton cache for manually registered singletons.
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
		// Check if bean definition exists in this factory.
		// 这里对IoC容器中的BeanDefinition是否存在进行检查，检查是否能在当前的BeanFactory中取得需要的Bean。
		// 如果当前的工厂中取不到，则到双亲BeanFactory中去取。如果当前的双亲工厂取不到，那就顺着双亲BeanFactory链一直向上查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				// 递归调用父bean的doGetBean查找
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}
		try {
			//这里根据Bean的名字取得BeanDefinition
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);
			// Guarantee initialization of beans that the current bean depends on.
			// 获取当前Bean的所有依赖Bean,这里会触发getBean的递归调用.知道取到一个没有任何依赖的Bean为止.
			// 这里的依赖和依赖注入不是一个概念,要注意,此处是bean创建的顺序依赖
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					registerDependentBean(dep, beanName);
					try {
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}
			// Create bean instance.
			// 这里通过createBean方法创建singleton Bean的实例
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						// 在getSingleton中会调用这个方法
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						// Explicitly remove instance from singleton cache: It might have been put there
						// eagerly by the creation process, to allow for circular reference resolution.
						// Also remove any beans that received a temporary reference to the bean.
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
			// 这里是创建prototype bean的地方
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}
	// Check if required type matches the type of the actual bean instance.
	//这里对创建的Bean进行类型检查，如果没有问题，就返回这个新创建的Bean，这个Bean已经是包含了依赖关系的Bean
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

上面代码中,我们主要来看一下if (mbd.isSingleton()){...}分支代码段,这里是创建单例bean的核心逻辑.  
主要就分为两处:
*	getSingleton()
*	getObjectForBeanInstance()


###	三.1		sharedInstance = getSingleton()

```
sharedInstance = getSingleton(beanName, () -> {
	try {
		return createBean(beanName, mbd, args);
	}
	catch (BeansException ex) {
		// Explicitly remove instance from singleton cache: It might have been put there
		// eagerly by the creation process, to allow for circular reference resolution.
		// Also remove any beans that received a temporary reference to the bean.
		destroySingleton(beanName);
		throw ex;
	}
});
```

这里再调用getSingleton方法时,通过匿名内部类方式传入了一个ObjectFactory<?>实例,实现了ObjectFactory<?>接口的getObject()方法.


跟进getSingleton方法内部看一下:

```
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
```

这段代码的核心方法是singletonObject = singletonFactory.getObject();  
而此处singletonFactory.getObject()则会调用前面讲的匿名内部类实现,也就是createBean()方法.  
createBean()方法是关键逻辑,后面会详细说明.


###	三.2		getObjectForBeanInstance()

```
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
	String currentlyCreatedBean = this.currentlyCreatedBean.get();
	if (currentlyCreatedBean != null) {
		registerDependentBean(beanName, currentlyCreatedBean);
	}
	return super.getObjectForBeanInstance(beanInstance, name, beanName, mbd);
}
```

跟进到父类getObjectForBeanInstance方法中

```
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
	// Don't let calling code try to dereference the factory if the bean isn't a factory.
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		if (beanInstance instanceof NullBean) {
			return beanInstance;
		}
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
		}
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		return beanInstance;
	}
	// Now we have the bean instance, which may be a normal bean or a FactoryBean.
	// If it's a FactoryBean, we use it to create a bean instance, unless the
	// caller actually wants a reference to the factory.
	if (!(beanInstance instanceof FactoryBean)) {
		return beanInstance;
	}
	Object object = null;
	if (mbd != null) {
		mbd.isFactoryBean = true;
	}
	else {
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
		// Return bean instance from factory.
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// Caches object obtained from FactoryBean if it is a singleton.
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```

这里主要做下面几件事:  

1.	判断beanInstance是否是FactoryBean类型,如果不是的话就直接将beanInstance返回了.
2.	如果是FactoryBean类型的话,那么则会通过FactoryBean接口的getObject()方法来创建bean实例.

这里稍微提一下FactoryBean,这个接口是Spring提供的一个扩展点,很多框架都通过这个接口类来实现自己框架中一些实例对象的创建以及通过代码编码方式对这些实例对象进行AOP代理. 

具体的这里就不细说了,可自行查阅相关资料


###	三.3		createBean

这里先给出createBean的总体过程:

1.	Bean实例的创建
2.	Bean实例的初始化(依赖注入等)
3.	调用各初始化方法


```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;
	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}
	// Prepare method overrides.
	// 准备override的方法
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}
	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		// createBean之前调用BeanPostProcessor的postProcessBeforeInitialization和postProcessAfterInitialization方法
		// 默认不做任何处理所以会返回null
		// 但是如果我们重写了这两个方法,那么bean的创建过程就结束了,这里就为以后的annotation自动注入提供了钩子
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}
	try {
		// 实际执行createBean的是doCreateBean()方法
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isTraceEnabled()) {
			logger.trace("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		// A previously detected exception with proper bean creation context already,
		// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}
```

继续往下看doCreateBean方法:

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
		throws BeanCreationException {
	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		// 这里是创建Bean的地方，由createBeanInstance完成
		// 完成Bean初始化过程的第一步:创建实例
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}
	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				//	这里是前面三.1中所说的注册的那些内部BeanPostProcessor实例进行切入操作的时机
				//  注意一点,这里能进行切入执行的BeanPostProcessor是MergedBeanDefinitionPostProcessor的子类
				//	而CommonAnnotationBeanPostProcessor和AutowiredAnnotationBeanPostProcessor就是MergedBeanDefinitionPostProcessor的子类
				//	因此@Resource和@Autowired注解的解析就是在这里做的(当然,还包括一些其他注解的处理)
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}
	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
	// 是否自动解决循环引用
	// 当bean条件为： 单例&&允许循环引用&&正在创建中  这样的话提早暴露一个ObjectFactory
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		// 把ObjectFactory放进singletonFactories中
		// 这里在其他bean在创建的时候会先去singletonFactories中查找有没有beanName到ObjectFactory的映射
		// 如果有ObjectFactory就调用它的getObject方法获取实例
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}
	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		// TODO 完成Bean初始化过程的第二步：为Bean的实例设置属性
		// Bean依赖注入发生的地方
		// 对bean进行属性填充，如果存在依赖于其他的bean的属性，则会递归的调用初始化依赖的bean
		populateBean(beanName, mbd, instanceWrapper);

		// TODO 完成Bean初始化过程的第三步：调用Bean的初始化方法（init-method）
		// 调用初始化方法，比如init-method方法指定的方法
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}
	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}
	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}
	return exposedObject;
}
```

将上述代码段中的核心方法对应到createBean的主体过程下即是:

1.	Bean实例的创建,instanceWrapper = createBeanInstance(beanName, mbd, args);
2.	Bean实例的初始化(依赖注入等)	,populateBean(beanName, mbd, instanceWrapper);
3.	调用各初始化方法,exposedObject = initializeBean(beanName, exposedObject, mbd);

下面,就按着这三个关键步骤,逐个分析相应的方法逻辑.


####	三.3.1	Bean实例的创建,instanceWrapper = createBeanInstance(beanName, mbd, args);

```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
	// 确保此时需要创建的Bean的class类型已经被解析.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}
	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}
	// 当有工厂方法的时候使用工厂方法初始化Bean,就是配置的时候指定FactoryMethod属性,
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			// 如果有有参数的构造函数,构造函数自动注入
			// 这里spring会花费大量的精力去进行参数的匹配
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			// 如果没有有参构造函数，使用默认构造函数构造
			return instantiateBean(beanName, mbd);
		}
	}

	// 使用带参数构造函数进行实例化
	// Candidate constructors for autowiring?
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		return autowireConstructor(beanName, mbd, ctors, args);
	}
	// Preferred constructors for default construction?
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}

	// 使用默认的构造函数对Bean进行实例化
	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
```

我们可以看到这里生成了Bean所包含的Java对象,这个对象的生成有很多种不同的方式,可以通过工厂方法生成,也可以通过容器的autowire特性生成,这些生成方式都是由BeanDefinition决定的.

我们再以instantiateBean(beanName, mbd)为例,跟进看下其内部实现

```
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
	// 使用默认的实例化策略对Bean进行实例化,默认的实例化策略是CglibSubclassingInstantiationStrategy
	try {
		Object beanInstance;
		final BeanFactory parent = this;
		if (System.getSecurityManager() != null) {
			beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
					getInstantiationStrategy().instantiate(mbd, beanName, parent),
					getAccessControlContext());
		}
		else {
			// getInstantiationStrategy()会返回CglibSubclassingInstantiationStrategy类的实例
			beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
		}
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
	}
}
```

可以看到getInstantiationStrategy().instantiate处的调用,典型的策略模式的设计模式.

这里使用CGLIB进行Bean的实例化.CGLIB是一个常用的字节码生成器的类库,其提供了一系列的API来提供生成和转换Java字节码的功能.在Spring AOP中同样也是可以使用CGLIB对Java的字节码进行增强.

在IoC容器中,使用SimpleInstantiationStrategy类.这个类是Spring用来生成Bean对象的默认类,它提供了两种实例化Java对象的方法,一种是通过BeanUtils,它使用的是JVM的反射功能,一种是通过CGLIB来生成.

getInstantiationStrategy()方法获取到CglibSubclassingInstantiationStrategy实例,instantiate()是CglibSubclassingInstantiationStrategy类的父类SimpleInstantiationStrategy实现的.


继续跟进到CglibSubclassingInstantiationStrategy的父类SimpleInstantiationStrategy中,看下instantiate方法的实现:

```
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
	// 如果BeanDefinition中没有包含了重写的方法(也即bean没有需要动态替换的方法),则使用CGLIB,否则使用BeanUtils(也就是java反射)
	if (!bd.hasMethodOverrides()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(
								(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
					}
					else {
						constructorToUse = clazz.getDeclaredConstructor();
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		// 通过BeanUtils进行实例化,这个BeanUtils的实例化通过Constructor类实例化Bean
		// 在BeanUtils中可以看到具体的调用ctor.newInstances(args)
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// Must generate CGLIB subclass.
		// TODO 使用CGLIB实例化对象
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```

在SpringBoot中我们一般采用@Autowire的方式进行依赖注入,很少采用像SpringMVC那种在xml中使用\<lookup-method\>或者\<replaced-method\>等标签的方式对注入的属性进行override,所以在上面的代码中if(!bd.hasMethodOverrides())中的判断为true,会采用BeanUtils的实例化方式


####	三.1.2	Bean实例的初始化(依赖注入等)	,populateBean(beanName, mbd, instanceWrapper);

```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	if (bw == null) {
		if (mbd.hasPropertyValues()) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
		}
		else {
			// Skip property population phase for null instance.
			return;
		}
	}
	// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
	// state of the bean before properties are set. This can be used, for example,
	// to support styles of field injection.
	boolean continueWithPropertyPopulation = true;
	// 调用InstantiationAwareBeanPostProcessor  Bean的后置处理器,在Bean注入属性前改变BeanDefinition的信息
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					continueWithPropertyPopulation = false;
					break;
				}
			}
		}
	}
	if (!continueWithPropertyPopulation) {
		return;
	}
	// 这里取得在BeanDefinition中设置的property值，这些property来自对BeanDefinition的解析
	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
	int resolvedAutowireMode = mbd.getResolvedAutowireMode();
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable.
		// A. 这里对autowire注入的处理，autowire by name
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
		// B. 这里对autowire注入的处理， autowire by type
		if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
	PropertyDescriptor[] filteredPds = null;
	if (hasInstAwareBpps) {
		if (pvs == null) {
			pvs = mbd.getPropertyValues();
		}
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
				if (pvsToUse == null) {
					if (filteredPds == null) {
						filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
					}
					// C. @Autowire @Resource @Value @Inject 等注解的依赖注入过程
					pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						return;
					}
				}
				pvs = pvsToUse;
			}
		}
	}
	if (needsDepCheck) {
		if (filteredPds == null) {
			filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		}
		checkDependencies(beanName, mbd, filteredPds, pvs);
	}
	if (pvs != null) {
		// 注入PropertyValues中的属性
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}
```

这里说一下几个关键点:  
1.	A处代码,和B处代码的applyPropertyValues()方法基本都是用于SpringMVC中采用xml配置Bean的方法,这里就不在深入介绍了,只需知道一点,xml解析bean的时候,是将xml中配置的属性依赖解析到BeanDefinition中的PropertyValues,然后再依赖PropertyValues中的数据来完成依赖注入的,而注解方式的依赖注入则不是.
2.	C处代码pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);这行是真正采用@Autowire @Resource @Value @Inject 等注解的依赖注入过程.


这里以AutowiredAnnotationBeanPostProcessor为例,跟进C处代码:

```
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	// 遍历,获取@Autowire,@Resource,@Value,@Inject等具备注入功能的注解
	InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
	try {
		// 属性注入
		metadata.inject(bean, beanName, pvs);
	}
	catch (BeanCreationException ex) {
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
	}
	return pvs;
}
```

AutowiredAnnotationBeanPostProcessor类实现了postProcessPropertyValues()方法.findAutowiringMetadata(beanName, bean.getClass(), pvs);方法会寻找在当前类中的被@Autowire,@Resource,@Value,@Inject等具备注入功能的注解的属性.


再跟进metadata.inject(bean, beanName, pvs);代码看下内部实现:

```
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Collection<InjectedElement> checkedElements = this.checkedElements;
	Collection<InjectedElement> elementsToIterate =
			(checkedElements != null ? checkedElements : this.injectedElements);
	if (!elementsToIterate.isEmpty()) {
		for (InjectedElement element : elementsToIterate) {
			if (logger.isTraceEnabled()) {
				logger.trace("Processing injected element of bean '" + beanName + "': " + element);
			}
			element.inject(target, beanName, pvs);
		}
	}
}
```

继续跟进element.inject(target, beanName, pvs);这一行

```
// AutowiredAnnotationBeanPostProcessor类
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	// 需要注入的字段
	Field field = (Field) this.member;
	// 需要注入的属性值
	Object value;
	if (this.cached) {
		value = resolvedCachedArgument(beanName, this.cachedFieldValue);
	}
	else {
		// @Autowired(required = false),当在该注解中设置为false的时候,如果有直接注入,没有跳过,不会报错
		DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
		desc.setContainingClass(bean.getClass());
		Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
		Assert.state(beanFactory != null, "No BeanFactory available");
		TypeConverter typeConverter = beanFactory.getTypeConverter();
		try {
			// 通过BeanFactory 解决依赖关系
			// 比如在webController中注入了webService,这个会去BeanFactory中去获取webService,也就是getBean()的逻辑
			// 如果存在直接返回,不存在再执行createBean()逻辑
			// 如果在webService中依然有依赖,依然会去继续递归
			// 这里是一个复杂的递归逻辑
			value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
		}
		synchronized (this) {
			if (!this.cached) {
				if (value != null || this.required) {
					this.cachedFieldValue = desc;
					registerDependentBeans(beanName, autowiredBeanNames);
					if (autowiredBeanNames.size() == 1) {
						String autowiredBeanName = autowiredBeanNames.iterator().next();
						if (beanFactory.containsBean(autowiredBeanName) &&
								beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
							this.cachedFieldValue = new ShortcutDependencyDescriptor(
									desc, autowiredBeanName, field.getType());
						}
					}
				}
				else {
					this.cachedFieldValue = null;
				}
				this.cached = true;
			}
		}
	}
	if (value != null) {
		ReflectionUtils.makeAccessible(field);
		field.set(bean, value);
	}
}
```
resolveDependency()方法中经过一顿操作，最终又会来到上面的getBean()方法(递归处理)  
其中resolveDependency()方法内部会调用doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter),在这一方法内部完成了依赖注入更底层的逻辑.这里就不再继续深挖了.


####	三.1.3	调用各初始化方法,exposedObject = initializeBean(beanName, exposedObject, mbd);

```
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			invokeAwareMethods(beanName, bean);
			return null;
		}, getAccessControlContext());
	}
	else {
		//在调用Bean的初始化方法之前,调用一系列的aware接口实现,把相关的BeanName,BeanClassLoader,以及BeanFactory注入到Bean中去
		invokeAwareMethods(beanName, bean);
	}
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		// 这些都是钩子方法,在反复的调用,给Spring带来了极大的可拓展性
		// 初始化之前调用BeanPostProcessor
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}
	try {
		// 调用指定的init-method方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
		// 这些都是钩子方法,在反复的调用,给Spring带来了极大的可拓展性
		// 初始化之后调用BeanPostProcessor
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

在跟进invokeInitMethods方法内部看下:

```
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
		throws Throwable {
	// 除了使用init-method指定的初始化方法,还可以让bean实现InitializingBean接口重写afterPropertiesSet()方法
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
		}
		if (System.getSecurityManager() != null) {
			try {
				AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
					((InitializingBean) bean).afterPropertiesSet();
					return null;
				}, getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
			// 执行afterPropertiesSet()方法进行初始化
			((InitializingBean) bean).afterPropertiesSet();
		}
	}
	// 先执行afterPropertiesSet()方法,再进行init-method
	if (mbd != null && bean.getClass() != NullBean.class) {
		String initMethodName = mbd.getInitMethodName();
		if (StringUtils.hasLength(initMethodName) &&
				!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
```

该方法中首先判断Bean是否配置了init-method方法,如果有,那么通过invokeCustomInitMethod()方法来直接调用.其中在invokeCustomInitMethod()方法中是通过JDK的反射机制得到method对象,然后调用的init-method.最终完成Bean的初始化.



到这里,bean创建,依赖注入和初始化方法完成