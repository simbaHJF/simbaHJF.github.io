---
layout:     post
title:      "springboot启动流程----三级缓存"
date:       2021-05-30 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - spring

---





## 导航
[一. 三级缓存数据结构](#jump1)
<br>
[二. 三级缓存查询逻辑](#jump2)
<br>
[三. getBean整体流程](#jump3)
<br>








<br><br>
## <span id="jump1">一. 三级缓存数据结构</span>

```
/** Cache of singleton objects: bean name to bean instance. */
// 第一级缓存,用于存放完全初始化好的 bean,从该缓存中取出的 bean 可以直接使用
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name to ObjectFactory. */
// 第三级缓存,单例对象工厂的cache,存放 bean 工厂对象,用于解决循环依赖
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name to bean instance. */
// 第二级缓存,提前曝光的单例对象的cache,存放原始的 bean 对象(尚未填充属性),用于解决循环依赖
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```



<br><br>
## <span id="jump2">二. 三级缓存查询逻辑</span>

源码如下:
```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

流程如下:
1. 查询第一级缓存,若存在,则直接返回
2. 查询第二级缓存,若存在,则直接返回
3. 查询第三级缓存,若存在通过其对应objectFactory的getObject()方法获取bean实例,放入第二级缓存,从第三级缓存中删除,返回bean实例
    > 此时返回的bean实例还并不完整,还未进行属性填充依赖注入和初始化,但已完成了实例化,已可用,至此可解决循环依赖问题



<br><br>
## <span id="jump3">三. getBean三级缓存整体流程</span>

[![2VJ8MD.png](https://z3.ax1x.com/2021/05/30/2VJ8MD.png)](https://imgtu.com/i/2VJ8MD)