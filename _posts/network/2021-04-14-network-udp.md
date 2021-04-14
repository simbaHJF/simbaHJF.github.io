---
layout:     post
title:      "UDP"
date:       2021-04-14 10:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 网络

---



## 导航
[一. UDP概述](#jump1)
<br>
[二. UDP的首部格式](#jump2)
<br>








<br><br>
## <span id="jump1">一. UDP概述</span>

UDP协议只在IP数据报服务至上增加了很少的一点功能,这就是复用和分用以及差错检测功能,UDP特点是:
1. UDP是无连接的
2. UDP使用尽最大努力交付,不保证可靠交付,因此不需要维持复杂的连接状态
3. UDP面向报文.发送方的UDP对应用程序交下来的报文,在添加首部后就向下交付IP层.UDP对应用层交下来的报文既不合并,也不拆分,而是保留这些报文的边界.
4. UDP没有拥塞控制
5. UDP支持一对一,一对多,多对一和多对多的交互通信
6. UDP的首部开销小,只有8个字节,比TCP的20个字节的首部要短



<br><br>
## <span id="jump2">二. UDP的首部格式</span>

用户数据报UDP有两个字段: 数据字段和首部字段,首部字段共8个字节,由4部分组成,每部分都是两字节:
* 源端口
* 目的端口
* 长度 : UDP用户数据报的长度
* 校验和 : 检测UDP用户数据报在传输中是否有错,有错就丢弃

[![c6CFZq.png](https://z3.ax1x.com/2021/04/14/c6CFZq.png)](https://imgtu.com/i/c6CFZq)

当运输层从IP层收到UDP数据报时,就根据首部中的目的端口,把UDP数据报通过相应的端口,上交最后的重点----应用进程.
[![c6PQ1g.png](https://z3.ax1x.com/2021/04/14/c6PQ1g.png)](https://imgtu.com/i/c6PQ1g)