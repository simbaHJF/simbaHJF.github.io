---
layout:     post
title:      "springboot一些总结"
date:       2021-05-30 16:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---




<br>
**<font size="4">1</font>** <br>

非Springboot项目一般使用XmlWebApplicationContext,Springboot项目一般使用AnnotationConfigServletWebServerApplicationContext<br>


<br>
**<font size="4">2</font>** <br>

非Springboot项目XmlWebApplicationContext在refresh时,是在obtainFreshBeanFactory时完成对xml配置文件中的bean进行BeanDefinition解析载入的<br>

SpringBoot项目BeanDefinition的解析则是由ConfigurationClassPostProcessor完成的,它实现了BeanDefinitionRegistryPostProcessor接口,调用postProcessBeanDefinitionRegistry完成BeanDefinition解析,而BeanDefinitionRegistryPostProcessor继承BeanFactoryPostProcessor<br>


<br>
**<font size="4">3</font>** <br>

非Springboot项目,解析载入BeanDefinition时,将依赖解析为PropertyValue,在后面依赖注入时,根据该数据进行依赖注入<br>

非Springboot项目,是依赖AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor完成依赖解析的,将依赖解析为InjectionMetadata,解析时机在bean实例化之后,属性填充populateBean之前<br>


<br>
**<font size="4">4</font>** <br>

BeanFacotry是spring中比较原始的Factory.如XMLBeanFactory就是一种典型的BeanFactory.原始的BeanFactory无法支持spring的许多插件,如AOP功能、Web应用等. <br>
ApplicationContext接口,它由BeanFactory接口派生而来,ApplicationContext包含BeanFactory的所有功能,通常建议比BeanFactory优先<br>

BeanFactory和FactoryBean的区别:
* BeanFactory是接口，提供了IOC容器最基本的形式，给具体的IOC容器的实现提供了规范，
* FactoryBean也是接口，为IOC容器中Bean的实现提供了更加灵活的方式，FactoryBean在IOC容器的基础上给Bean的实现加上了一个简单工厂模式和装饰模式,我们可以在getObject()方法中灵活配置.其实在Spring源码中有很多FactoryBean的实现类.

区别:BeanFactory是个Factory,也就是IOC容器或对象工厂,FactoryBean是个Bean.在Spring中,所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的.但对FactoryBean而言,这个Bean不是简单的Bean,而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似.


<br>
**<font size="4">5</font>** <br>

application.properties配置加载到Environment中的机制<br>

通过SPI引入的启动时相关监听器:ConfigFileApplicationListener<br>

ConfigFileApplicationListener会监听ApplicationEnvironmentPreparedEvent事件,同时ConfigFileApplicationListener是EnvironmentPostProcessor的实现类,事件触发后会调用postProcessEnvironment方法,该方法内部完成对配置文件kv属性的读取,注入environment中<br>



<br>
**<font size="4">6</font>** <br>

spring-boot内置tomcat原理<br>

项目推断为web类型的,会生成类型为AnnotationConfigServletWebServerApplicationContext的上下文.该上下文类下的onRefresh方法内部,会通过调用tomcat的api接口,配置并启动tomcat.<br>



<br>
**<font size="4">7</font>** <br>

BeanDefinition的加载解析原理<br>

AnnotationConfigServletWebServerApplicationContext构造方法中会创建AnnotatedBeanDefinitionReader对象,创建AnnotatedBeanDefinitionReader对象底层会注册一个ConfigurationClassPostProcessor的BeanDefinition.这个类是BeanDefinitionRegistryPostProcessor接口的一个实现类,而BeanDefinitionRegistryPostProcessor接口继承了BeanFactoryPostProcessor接口,因此会在AbstractApplicationContext的refresh模板方法中的invokeBeanFactoryPostProcessors方法内部被调用,然后被调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法来完成BeanDefinition的加载和解析<br>



<br>
**<font size="4">8</font>** <br>

依赖注入原理: CommonAnnotationBeanPostProcessor 和 AutowiredAnnotationBeanPostProcessor<br>

都是InstantiationAwareBeanPostProcessor接口的实现类,在populateBean方法中,会调用InstantiationAwareBeanPostProcessor接口的postProcessPropertyValues方法,完成依赖注入<br>



<br>
**<font size="4">9</font>** <br>

自动装配原理:<br>

BeanDefinition扫描时,ConfigurationClassPostProcessor会处理@Import,主类@SpringBootApplication注解内部引入@EnableAutoConfiguration注解,从而引入AutoConfigurationImportSelector. AutoConfigurationImportSelector类的方法内部逻辑,会通过Spring的SPI机制引入spring.factories文件下org.springframework.boot.autoconfigure.EnableAutoConfiguration的各个实现类,从而引入了要装配的配置类.然后通过SPI机制,引入Filter链,通过Filter链对前面的自动配置类进行过滤,获取到有效的(@ConditionalOnBean等这些条件装配,就是在这个Filter链中处理的)<br>

