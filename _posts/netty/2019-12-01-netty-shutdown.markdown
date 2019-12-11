---
layout:     post
title:      "netty shutdown"
date:       2019-11-27 23:40:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

#	主线

*	bossGroup.shtudownGracefully();<br>
	workGroup.shutdownGracefully();<br>
	关闭所有Group中的NioEventLoop:
	*	修改NioEventLoop的State标志位
	*	NioEventLoop判断State执行退出



![QeWiVI.jpg](https://s2.ax1x.com/2019/12/01/QeWiVI.jpg)