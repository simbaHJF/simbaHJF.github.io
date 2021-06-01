---
layout:     post
title:      "springboot aop"
date:       2021-05-31 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---








## 导航
[一. 配置类的引入](#jump1)
<br>
[二. AnnotationAwareAspectJAutoProxyCreator类分析](#jump2)
<br>











实际项目中,使用AOP最常见的形式,是基于AspectJ注解的实现形式.本文也基于这个前提来进行相关分析.<br>



<br><br>
## <span id="jump1">一. 配置类的引入</span>

[![2m9EP1.png](https://z3.ax1x.com/2021/05/31/2m9EP1.png)](https://imgtu.com/i/2m9EP1)

由spring.factories文件可以看出,引入了AopAutoConfiguration这一自动配置类.下面继续跟进AopAutoConfiguration配置类,这里我们值关注AspectJ方式相关的,也即下图中的红色部分.

[![2minMV.png](https://z3.ax1x.com/2021/05/31/2minMV.png)](https://imgtu.com/i/2minMV)

不论是JdkDynamicAutoProxyConfiguration配置类,还是CglibAutoProxyConfiguration配置类,都是引入@EnableAspectJAutoProxy来实现的. 下面跟进其内部
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    /**
     * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
     * to standard Java interface-based proxies. The default is {@code false}.
     */
    boolean proxyTargetClass() default false;
    /**
     * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
     * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
     * Off by default, i.e. no guarantees that {@code AopContext} access will work.
     * @since 4.3.1
     */
    boolean exposeProxy() default false;
}
```

可见,其功能实现是依赖于引入AspectJAutoProxyRegistrar类来实现,继续往下跟进

```
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     * Register, escalate, and configure the AspectJ auto proxy creator based on the value
     * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
     * {@code @Configuration} class.
     */
    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // 1. 注册AnnotationAwareAspectJAutoProxyCreator的BeanDefinition
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy =
                AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}
```

这里重点关注1处标注的方法:
```
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
        BeanDefinitionRegistry registry, @Nullable Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}


@Nullable
private static BeanDefinition registerOrEscalateApcAsRequired(
        Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```

至此,可以知道,配置类底层最核心的功能就是向容器,也即registry中注册了一个AnnotationAwareAspectJAutoProxyCreator类的BeanDefinition.<br>



<br><br>
## <span id="jump2">二. AnnotationAwareAspectJAutoProxyCreator类分析</span>

先来看下AnnotationAwareAspectJAutoProxyCreator类的继承关系
[![2mT1HK.png](https://z3.ax1x.com/2021/06/01/2mT1HK.png)](https://imgtu.com/i/2mT1HK)

可见AnnotationAwareAspectJAutoProxyCreator类实现了BeanPostProcessor接口,那么就好办了,接下来就可以分析其相应的切入方法了.<br>

来看下它对postProcessAfterInitialization方法的实现
```
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}


protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // Create proxy if we have advice.
    // 1.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

wrapIfNecessary方法为核心方法,其核心流程如下:
* 获取当前bean所适用的advisor列表
* 如果拦截器列表为空,说明当前bean不需要进行aop增强,那么直接返回原始bean
* 如果拦截器列表不为空,说明当前bean需要进行aop增强,那么为它生成代理对象返回,生成代理对象的方式无非就是jdk proxy或者是cglib proxy两种方式,根据bean的情况具体判断.

而getAdvicesAndAdvisorsForBean用于获取适用的advisor列表,其内部逻辑就是首先获取所有的candidateAdvisors,然后遍历,用每一个Advisor和被代理类中的所有方法进行匹配,只要有一个匹配到,就意味着该类应该被代理,否则该类不需要被代理.<br>

接下来就是通过createProxy方法来生成代理类了,深入方法内部,关键方法如下:
```
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

proxyTargetClass在前面配置类AopAutoConfiguration中指定,默认为true.因此,这个方法整体只有一种情况会走jdk代理方法,就是代理类为接口类型(<font color="red">注意,代理类是接口,并不是指该类是否实现了接口</font>)或者代理类是Proxy类型,否则全部走cglib代理.所以,在平时使用时,代理类大部分还是用cglib的方式来生成的.<br>

JdkDynamicAopProxy实现了InvocationHandler接口,其核心逻辑在于invoke方法;CglibAopProxy中有内部类DynamicAdvisedInterceptor,DynamicAdvisedInterceptor实现了MethodInterceptor接口,其核心逻辑在于intercept方法.因此两种代理方式中,InvocationHandler接口可类比MethodInterceptor接口,对应的invoke方法类比intercept方法,内部逻辑是相似的.<br>


下面以CglibAopProxy代理方式简要分析下代理执行流程,首先放上DynamicAdvisedInterceptor的核心方法intercept的源码
```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();
    try {
        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        // Check whether we only have one InvokerInterceptor: that is,
        // no real advice, but just reflective invocation of the target.
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // We need to create a method invocation...
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

流程如下:
1. 获取代理逻辑责任链
2. 如果责任链为空,那么该方法不需要执行代理逻辑,直接调用源对象target的对应方法即可
3. 如果责任链不为空,那么该方法需要被代理,责任链方式执行代理方法及源对象target的对应方法.