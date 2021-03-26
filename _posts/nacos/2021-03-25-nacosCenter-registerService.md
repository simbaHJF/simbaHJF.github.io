---
layout:     post
title:      "Nacos注册中心:nacos-center服务注册"
date:       2021-03-25 15:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - nacos

---




## 导航
[一. nacos-center端服务注册信息数据结构管理](#jump1)
<br>













<br><br>
## <span id="jump1">一. nacos-center端服务注册信息数据结构管理</span>

**<font size="5">nacos-center端,对服务注册信息的管理,最顶层由ServiceManager统一进行管理</font>**
[![6OQeJI.png](https://z3.ax1x.com/2021/03/25/6OQeJI.png)](https://imgtu.com/i/6OQeJI)

**<font size="5">Service对服务注册信息管理的数据结构</font>**
[![6OQvnS.png](https://z3.ax1x.com/2021/03/25/6OQvnS.png)](https://imgtu.com/i/6OQvnS)

**<font size="5">Cluster对服务注册信息管理的数据结构</font>**
[![6O1PVe.png](https://z3.ax1x.com/2021/03/25/6O1PVe.png)](https://imgtu.com/i/6O1PVe)

**<font size="5">整体数据结构示意图</font>**
[![6O1zJs.png](https://z3.ax1x.com/2021/03/25/6O1zJs.png)](https://imgtu.com/i/6O1zJs)



<br><br>
## <span id="jump2">二. nacos-center,服务注册流程</span>

流程图转自

> https://blog.csdn.net/weixin_42437633/article/details/105336354?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control


[![6jY7V0.png](https://z3.ax1x.com/2021/03/26/6jY7V0.png)](https://imgtu.com/i/6jY7V0)