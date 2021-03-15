---
layout:     post
title:      "redis sentinel"
date:       2021-03-15 23:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - redis

---



## 导航
[一. 前言](#jump1)
<br>











<br><br>
## <span id="jump1">前言</span>

Sentinel(哨兵)是Redis的高可用解决方案:由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器,以及这些主服务器属下的所有从服务器,并在被监视的主服务器进入下线状态时,自动将下线主服务器属下的某个从服务器升级为新的主服务器,然后由新的主服务器代替已下线的主服务器继续处理命令请求.

[![6rUnjU.png](https://s3.ax1x.com/2021/03/15/6rUnjU.png)](https://imgtu.com/i/6rUnjU)
[![6rUY36.png](https://s3.ax1x.com/2021/03/15/6rUY36.png)](https://imgtu.com/i/6rUY36)
[![6rU4Ej.png](https://s3.ax1x.com/2021/03/15/6rU4Ej.png)](https://imgtu.com/i/6rU4Ej)
[![6rUI5n.png](https://s3.ax1x.com/2021/03/15/6rUI5n.png)](https://imgtu.com/i/6rUI5n)

当server1的下线时长超过用户设定的下线时长上限时,Sentinel系统就会对server1执行故障转移操作:
* 首先,Sentinel系统会挑选server1属下的的其中一个从服务器,并将这个被选中的从服务器升级为新的主服务器.
* 之后,Sentinel系统会向server1属下的所有从服务器发送新的复制指令,让它们成为新的主服务器的从服务器,当所有从服务器都开始复制新的主服务器时,故障转移操作执行完毕
* 另外,Sentinel还会继续监视已下线的server1,并在它重新上线时,将它设置为新的主服务器的从服务器