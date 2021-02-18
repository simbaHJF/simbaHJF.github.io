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
[一. RouterChain、RouterFactory 与 Router](#jump1)
<br>
[二. ConditionRouterFactory&ConditionRouter](#jump2)
<br>





Router 的主要功能就是根据用户配置的路由规则以及请求携带的信息,过滤出符合条件的 Invoker 集合,供后续负载均衡逻辑使用.



<br><br>
## <span id="jump1">一. RouterChain、RouterFactory 与 Router</span>

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

接下来我们就开始介绍 RouterFactory 以及 Router 的具体实现<br>



<br><br>
## <span id="jump2">二. ConditionRouterFactory&ConditionRouter</span>

首先来看 ConditionRouterFactory 实现,其扩展名为 condition,在其 getRouter() 方法中会创建 ConditionRouter 对象,如下所示
```
public Router getRouter(URL url) {
    return new ConditionRouter(url);
}
```

ConditionRouter 是基于条件表达式的路由实现类,下面就是一条基于条件表达式的路由规则:<br>
``
host = 192.168.0.100 => host = 192.168.0.150
``

在上述规则中,=>之前的为 Consumer 匹配的条件,该条件中的所有参数会与 Consumer 的 URL 进行对比,当 Consumer 满足匹配条件时,会对该 Consumer 的此次调用执行 => 后面的过滤规则.=> 之后为 Provider 地址列表的过滤条件,该条件中的所有参数会和 Provider 的 URL 进行对比,Consumer 最终只拿到过滤后的地址列表.<br>

如果 Consumer 匹配条件为空,表示 => 之后的过滤条件对所有 Consumer 生效,例如:=> host != 192.168.0.150,含义是所有 Consumer 都不能请求 192.168.0.150 这个 Provider 节点.<br>

如果 Provider 过滤条件为空,表示禁止访问所有 Provider,例如:host = 192.168.0.100 =>,含义是 192.168.0.100 这个 Consumer 不能访问任何 Provider 节点.<br>

ConditionRouter 的核心字段有如下几个
* url(URL 类型): 路由规则的 URL,可以从 rule 参数中获取具体的路由规则
* ROUTE_PATTERN(Pattern 类型): 用于切分路由规则的正则表达式
* priority(int 类型): 路由规则的优先级,用于排序,该字段值越大,优先级越高,默认值为 0
* force(boolean 类型): 当路由结果为空时,是否强制执行.如果不强制执行,则路由结果为空的路由规则将会自动失效;如果强制执行,则直接返回空的路由结果
* whenCondition(Map<String, MatchPair> 类型): Consumer 匹配的条件集合,通过解析条件表达式 rule 的 => 之前半部分,可以得到该集合中的内容
* thenCondition(Map<String, MatchPair> 类型): Provider 匹配的条件集合,通过解析条件表达式 rule 的 => 之后半部分,可以得到该集合中的内容

在 ConditionRouter 的构造方法中,会根据 URL 中携带的相应参数初始化 priority、force、enable 等字段,然后从 URL 的 rule 参数中获取路由规则进行解析,具体的解析逻辑是在 init() 方法中实现的,如下所示
```
public void init(String rule) {
    // 将路由规则中的"consumer."和"provider."字符串清理掉
    rule = rule.replace("consumer.", "").replace("provider.", "");
    // 按照"=>"字符串进行分割，得到whenRule和thenRule两部分
    int i = rule.indexOf("=>"); 
    String whenRule = i < 0 ? null : rule.substring(0, i).trim();
    String thenRule = i < 0 ? rule.trim() : rule.substring(i + 2).trim();
    // 解析whenRule和thenRule，得到whenCondition和thenCondition两个条件集合
    Map<String, MatchPair> when = StringUtils.isBlank(whenRule) || "true".equals(whenRule) ? new HashMap<String, MatchPair>() : parseRule(whenRule);
    Map<String, MatchPair> then = StringUtils.isBlank(thenRule) || "false".equals(thenRule) ? null : parseRule(thenRule);
    this.whenCondition = when;
    this.thenCondition = then;
}
```

whenCondition 和 thenCondition 两个集合中,Key 是条件表达式中指定的参数名称(例如 host = 192.168.0.150 这个表达式中的 host). ConditionRouter 支持三类参数:
* 服务调用信息,例如,method、argument 等
* URL 本身的字段,例如,protocol、host、port 等
* URL 上的所有参数,例如,application 等