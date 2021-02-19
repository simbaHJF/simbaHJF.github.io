---
layout:     post
title:      "Dubbo 路由机制"
date:       2021-02-16 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---



## 导航
[一. 路由机制的作用意义](#jump2)
<br>
[二. RouterChain、RouterFactory 与 Router](#jump2)
<br>




<br><br>
## <span id="jump1">一. 路由机制的作用意义</span>

Router 的主要功能就是根据用户配置的路由规则以及请求携带的信息,过滤出符合条件的 Invoker 集合,供后续负载均衡逻辑使用.



<br><br>
## <span id="jump2">二. RouterChain、RouterFactory 与 Router</span>

首先我们来看 RouterChain 的核心字段
* invokers(List<Invoker<T>> 类型): 当前 RouterChain 对象要过滤的 Invoker 集合.我们可以看到,在 StaticDirectory 中是通过 RouterChain.setInvokers() 方法进行设置的
* builtinRouters(List<Router> 类型): 当前 RouterChain 激活的内置 Router 集合
* routers(List<Router> 类型): 当前 RouterChain 中真正要使用的 Router 集合,其中不仅包括了上面 builtinRouters 集合中全部的 Router 对象,还包括通过 addRouters() 方法添加的 Router 对象


在 RouterChain 的构造函数中,会在传入的 URL 参数中查找 router 参数值,并根据该值获取确定激活的 RouterFactory,之后通过 Dubbo SPI 机制加载这些激活的 RouterFactory 对象,由 RouterFactory 创建当前激活的内置 Router 实例,具体实现如下
```
private RouterChain(URL url) {
    // 通过ExtensionLoader加载激活的RouterFactory
    List<RouterFactory> extensionFactories = ExtensionLoader.getExtensionLoader(RouterFactory.class)
            .getActivateExtension(url, "router");
    // 遍历所有RouterFactory，调用其getRouter()方法创建相应的Router对象
    List<Router> routers = extensionFactories.stream()
            .map(factory -> factory.getRouter(url))
            .collect(Collectors.toList());
    initWithRouters(routers); // 初始化buildinRouters字段以及routers字段
}
public void initWithRouters(List<Router> builtinRouters) {
    this.builtinRouters = builtinRouters;
    this.routers = new ArrayList<>(builtinRouters);
    this.sort(); // 这里会对routers集合进行排序
}
```

完成内置 Router 的初始化之后,在 Directory 实现中还可以通过 addRouter() 方法添加新的 Router 实例到 routers 字段中,具体实现如下
```
public void addRouters(List<Router> routers) {
    List<Router> newRouters = new ArrayList<>();
    newRouters.addAll(builtinRouters); // 添加builtinRouters集合
    newRouters.addAll(routers); // 添加传入的Router集合
    CollectionUtils.sort(newRouters); // 重新排序
    this.routers = newRouters;
}
```

RouterChain.route() 方法会遍历 routers 字段,逐个调用 Router 对象的 route() 方法,对 invokers 集合进行过滤,具体实现如下:
```
public List<Invoker<T>> route(URL url, Invocation invocation) {
    List<Invoker<T>> finalInvokers = invokers;
    for (Router router : routers) { // 遍历全部的Router对象
        finalInvokers = router.route(finalInvokers, url, invocation);
    }
    return finalInvokers;
}
```

了解了 RouterChain 的大致逻辑之后,我们知道真正进行路由的是 routers 集合中的 Router 对象.接下来我们再来看 RouterFactory 这个工厂接口,RouterFactory 接口是一个扩展接口,具体定义如下:
```
@SPI
public interface RouterFactory {
    @Adaptive("protocol") // 动态生成的适配器会根据protocol参数选择扩展实现
    Router getRouter(URL url);
}
```

RouterFactory 接口有很多实现类,如下图所示:
[![yctzBq.png](https://s3.ax1x.com/2021/02/16/yctzBq.png)](https://imgchr.com/i/yctzBq)

下面我们就来深入介绍下每个 RouterFactory 实现类以及对应的 Router 实现对象.Router 决定了一次 Dubbo 调用的目标服务,Router 接口的每个实现类代表了一个路由规则,当 Consumer 访问 Provider 时,Dubbo 根据路由规则筛选出合适的 Provider 列表,之后通过负载均衡算法再次进行筛选.Router 接口的继承关系如下图所示:
[![ycaghT.png](https://s3.ax1x.com/2021/02/16/ycaghT.png)](https://imgchr.com/i/ycaghT)

有关 RouterFactory 以及 Router 的具体实现就翻源码吧.<br>


