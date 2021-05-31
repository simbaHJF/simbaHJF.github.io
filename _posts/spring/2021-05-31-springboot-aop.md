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
[![2mFo7D.png](https://z3.ax1x.com/2021/05/31/2mFo7D.png)](https://imgtu.com/i/2mFo7D)

可见AnnotationAwareAspectJAutoProxyCreator类实现了BeanPostProcessor接口,那么就好办了,接下来就可以分析其相应的切入方法了.