---
layout:     post
title:      "Dubbo Proxy层"
date:       2021-02-14 15:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---



## 导航
[一. Dubbo Proxy层的作用](#jump1)
<br>
[二. ProxyFactory](#jump2)
<br>
[三. 一点总结](#jump3)




<br><br>
## <span id="jump1">一. Dubbo Proxy层的作用</span>

Protocol 这一层以及后面介绍的 Cluster 层暴露出来的接口都是 Dubbo 内部的一些概念,业务层无法直接使用.为了让业务逻辑能够无缝使用 Dubbo,我们就需要将业务逻辑与 Dubbo 内部概念打通,这就用到了动态生成代理对象的功能.Proxy 层在 Dubbo 架构中的位置如下所示(虽然在架构图中 Proxy 层与 Protocol 层距离很远,但 Proxy 的具体代码实现就位于 dubbo-rpc-api 模块中)

[![yyIlIU.png](https://s3.ax1x.com/2021/02/14/yyIlIU.png)](https://imgchr.com/i/yyIlIU)



<br><br>
## <span id="jump2">二. ProxyFactory</span>

ProxyFactory 是一个扩展接口,其中定义了两个核心方法:
* 一个是 getProxy() 方法,为 Invoker 对象创建代理对象.用在服务引用端,将对Invoker创建代理对象,业务代码中使用的Service其实是这个代理对象.
* 另一个是 getInvoker() 方法,将代理对象反向封装成 Invoker 对象.对业务实现的Service创建代理对象为Invoker.

```
@SPI("javassist")
public interface ProxyFactory {
    // 为传入的Invoker对象创建代理对象
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;
    // 将传入的代理对象封装成Invoker对象
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```

看到 ProxyFactory 上的 @SPI 注解我们知道,其默认实现使用 Javassist 来创建代码对象
[![yyT5vj.png](https://s3.ax1x.com/2021/02/15/yyT5vj.png)](https://imgchr.com/i/yyT5vj)



<br><br>
## <span id="jump3">三. 一点总结</span>

Consumer 端的 Proxy 底层屏蔽了复杂的网络交互、集群策略以及 Dubbo 内部的 Invoker 等概念,提供给上层使用的是业务接口.Provider 端的 Wrapper 是将个性化的业务接口实现,统一转换成 Dubbo 内部的 Invoker 接口实现.正是由于 Proxy 和 Wrapper 这两个组件的存在,Dubbo 才能实现内部接口和业务接口的无缝转换.<br>

具体的代理源码这里就不展开分析了,查阅源码翻看吧.主要流程就如下图所示.

[![yy55vR.png](https://s3.ax1x.com/2021/02/14/yy55vR.png)](https://imgchr.com/i/yy55vR)