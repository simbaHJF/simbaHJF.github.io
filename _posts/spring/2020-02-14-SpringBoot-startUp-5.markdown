---
layout:     post
title:      "springboot启动流程----自动装配的实现"
date:       2020-02-14 20:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---

> springboot版本:2.2.1



#	一.	@SpringBootApplication注解注解

首先来看一下springboot项目主类上的@SpringBootApplication注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	......
}

```

这个注解引包含了注解@EnableAutoConfiguration

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ......
}
```

至此,我们看到通过@Import(AutoConfigurationImportSelector.class)导入了一个重要的类AutoConfigurationImportSelector.


#	二.	AutoConfigurationImportSelector

首先从springboot启动的源码过程来跟进AutoConfigurationImportSelector是如何被调用的.

前面讲springboot启动流程的BeanDefinition解析时,讲到过在ConfigurationClassPostProcessor的processConfigBeanDefinitions方法中完成解析.

该方法中会调用ConfigurationClassParser的parse方法来进行处理

这里先放上ConfigurationClassParser的parse方法的源码:

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

这里与自动装配有关的关键点有两个:
*	第一个条件判断分支中的parse方法,前面分析过这里面的BeanDefinition解析工作
*	this.deferredImportSelectorHandler.process();

我们来逐个分析.

##	二.1		parse方法

这里我们只跟进和自动装配有关的,其他部分和BeanDefinition解析有关的前面的文章已经讲过,这里就不再重复.  
从parse方法开始,沿着下面调用链路跟进追踪:  
parse方法 ----> processConfigurationClass方法 ----> doProcessConfigurationClass方法 ----> processImports

至此跟踪到如下代码段:
```
// Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), true);
```

###	二.1.1	getImports(sourceClass)

首先跟进getImports(sourceClass)方法内部:
```
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
	Set<SourceClass> imports = new LinkedHashSet<>();
	Set<SourceClass> visited = new LinkedHashSet<>();
	collectImports(sourceClass, imports, visited);
	return imports;
}
```

跟进collectImports(sourceClass, imports, visited);
```
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
		throws IOException {
	if (visited.add(sourceClass)) {
		for (SourceClass annotation : sourceClass.getAnnotations()) {
			String annName = annotation.getMetadata().getClassName();
			if (!annName.equals(Import.class.getName())) {
				collectImports(annotation, imports, visited);
			}
		}
		imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
	}
}
```

可以看到这里是递归调用获取所有的import引入类,一个典型的DFS深度优先遍历算法的使用.

经过上面处理getImports(sourceClass)方法的返回中将包含AutoConfigurationImportSelector
[![1x2xMj.png](https://s2.ax1x.com/2020/02/15/1x2xMj.png)](https://imgchr.com/i/1x2xMj)

###	二.1.2	processImports

回到processImports(configClass, sourceClass, getImports(sourceClass), true)方法;  
由前面分析我们得到,这时该方法的第三个参数也就是getImports(sourceClass)返回的Set中包含AutoConfigurationImportSelector.  
debug跟进进去,打断点,显示如下:
[![1xhlF0.png](https://s2.ax1x.com/2020/02/15/1xhlF0.png)](https://imgchr.com/i/1xhlF0)

再跟进this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);这一行

```
public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
	DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(
			configClass, importSelector);
	if (this.deferredImportSelectors == null) {
		DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
		handler.register(holder);
		handler.processGroupImports();
	}
	else {
		this.deferredImportSelectors.add(holder);
	}
}
```

至此,AutoConfigurationImportSelector就被添加到了deferredImportSelectors中.


##	二.2		this.deferredImportSelectorHandler.process();

回到开头的parse(Set<BeanDefinitionHolder> configCandidates)方法内部.第二个关键点就是this.deferredImportSelectorHandler.process();的调用  

跟进该方法
```
public void process() {
	List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
	this.deferredImportSelectors = null;
	try {
		if (deferredImports != null) {
			DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
			deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
			deferredImports.forEach(handler::register);
			handler.processGroupImports();
		}
	}
	finally {
		this.deferredImportSelectors = new ArrayList<>();
	}
}
```

###		二.2.1	deferredImports.forEach(handler::register)
首先看一下deferredImports.forEach(handler::register);这一行中handler::register的实现

```
public void register(DeferredImportSelectorHolder deferredImport) {
	Class<? extends Group> group = deferredImport.getImportSelector()
			.getImportGroup();
	DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
			(group != null ? group : deferredImport),
			key -> new DeferredImportSelectorGrouping(createGroup(group)));
	grouping.add(deferredImport);
	this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
			deferredImport.getConfigurationClass());
}
```

这段代码简单讲就是将DeferredImportSelector以DeferredImportSelector.Group的形式组织.而AutoConfigurationImportSelector则属于AutoConfigurationGroup.class,AutoConfigurationGroup是AutoConfigurationImportSelector的一个内部类.  
AutoConfigurationImportSelector实现了DeferredImportSelector接口,并重写了方法getImportGroup():
```
@Override
public Class<? extends Group> getImportGroup() {
	return AutoConfigurationGroup.class;
}
```

然后将DeferredImportSelector.Group封装成DeferredImportSelectorGrouping.来看下其构造方法:
```
DeferredImportSelectorGrouping(Group group) {
	this.group = group;
}
```
而handler中的groupings属性则是\<Object,DeferredImportSelectorGrouping\>结构的Map.


###		二.2.2	handler.processGroupImports()

上面的准备工作完成,回到外层,再跟进handler.processGroupImports();这行
[![1z2rnA.png](https://s2.ax1x.com/2020/02/15/1z2rnA.png)](https://imgchr.com/i/1z2rnA)

跟进grouping.getImports()这一行
[![3SSzKH.png](https://s2.ax1x.com/2020/02/15/3SSzKH.png)](https://imgchr.com/i/3SSzKH)

由上一小节的分析,AutoConfigurationImportSelector对应的DeferredImportSelector.Group类型为AutoConfigurationGroup,由此我们会进入AutoConfigurationImportSelector下的内部类AutoConfigurationGroup中的process方法.

经过层层调用,至此,我们终于进到了AutoConfigurationImportSelector类中的逻辑处理.此处就是自动装配的关键了,下面来详细分析.



#	三.	AutoConfigurationGroup的process方法

经过前面的层层调用,终于进入到我们引入的关键类AutoConfigurationImportSelector中了.

```
public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
	Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
			() -> String.format("Only %s implementations are supported, got %s",
					AutoConfigurationImportSelector.class.getSimpleName(),
					deferredImportSelector.getClass().getName()));
	AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
			.getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
	this.autoConfigurationEntries.add(autoConfigurationEntry);
	for (String importClassName : autoConfigurationEntry.getConfigurations()) {
		this.entries.putIfAbsent(importClassName, annotationMetadata);
	}
}
```

跟进getAutoConfigurationEntry方法
```
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
		AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = filter(configurations, autoConfigurationMetadata);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```

跟进List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
			getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
			+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```

至此,我们又看到了熟悉springboot中的SPI机制,根据类的全限定名去获取需要加载的类名,加载如下
[![3Skfoj.png](https://s2.ax1x.com/2020/02/15/3Skfoj.png)](https://imgchr.com/i/3Skfoj)

然后执行操作:
*	去重
*	去掉需要排除的配置类
*	进行过滤

前两步很清晰,没什么好讲的.这里稍微分析一下过滤方法filter,看下源码:
```
private List<String> filter(List<String> configurations, AutoConfigurationMetadata autoConfigurationMetadata) {
	long startTime = System.nanoTime();
	String[] candidates = StringUtils.toStringArray(configurations);
	boolean[] skip = new boolean[candidates.length];
	boolean skipped = false;
	for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
		invokeAwareMethods(filter);
		boolean[] match = filter.match(candidates, autoConfigurationMetadata);
		for (int i = 0; i < match.length; i++) {
			if (!match[i]) {
				skip[i] = true;
				candidates[i] = null;
				skipped = true;
			}
		}
	}
	if (!skipped) {
		return configurations;
	}
	List<String> result = new ArrayList<>(candidates.length);
	for (int i = 0; i < candidates.length; i++) {
		if (!skip[i]) {
			result.add(candidates[i]);
		}
	}
	if (logger.isTraceEnabled()) {
		int numberFiltered = configurations.size() - result.size();
		logger.trace("Filtered " + numberFiltered + " auto configuration class in "
				+ TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime) + " ms");
	}
	return new ArrayList<>(result);
}
```
这里主要是获取到各个AutoConfigurationImportFilter完成匹配过滤,再来看下获取到了那些filter,也就是getAutoConfigurationImportFilters()方法

```
protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
	return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
}
```

又看到了熟悉的springboot的SPI机制,话不多说,直接上图,看加载了那些filter
![3SZdjP.png](https://s2.ax1x.com/2020/02/16/3SZdjP.png)

以上就是完成条件注入所需的各个AutoConfigurationImportFilter



springboot的自动装配至此分析完成.<br>


这里总结一下大体过程就是:
1. 项目主类上会有@SpringBootApplication注解
2. @SpringBootApplication注解内部包含@EnableAutoConfiguration注解
3. @EnableAutoConfiguration注解中有@Import(AutoConfigurationImportSelector.class)这一import标记
4. 在BeanDefinition解析时会有切入点,调用到AutoConfigurationImportSelector内部类AutoConfigurationGroup的process方法
5. 在process方法中,底层通过SPI机制将各个spring.factories文件中EnableAutoConfiguration的实现类全限定名获取到,然后将其放入configurationClasses中,表示配置类
6. ConfigurationClassPostProcessor在进行parse处理时,会将configurationClasses中的各个类解析为BeanDefinition注册到容器中
7. 各个外部starter的配置类至此可以生效了.