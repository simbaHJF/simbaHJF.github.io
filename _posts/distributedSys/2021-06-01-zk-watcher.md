---
layout:     post
title:      "zookeeper watcher机制"
date:       2021-06-01 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 分布式

---





[![2u6tc8.png](https://z3.ax1x.com/2021/06/01/2u6tc8.png)](https://imgtu.com/i/2u6tc8)


watcher机制特点: 一次触发性(也即一次失效性) <br>

鉴于此,client若想一直监听某个节点,需要在每次watcher触发通知中,再次注册watcher.<br>

正式因为这个鸟机制,zk并不适合在大量监听下,频繁变更数据.<br>