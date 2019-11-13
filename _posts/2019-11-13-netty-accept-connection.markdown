---
layout:     post
title:      "netty接收并构建连接"
date:       2019-11-12 23:15:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---


#	主线概述

###	boss thread

*	NioEventLoop中的selector轮询创建连接事件(OP_ACCEPT)
*	创建socket channel
*	初始化socket channel并从worker group中选择一个NioEventLoop


###	worker thread

*	将socket channel注册到选择的NioEventLoop的selector
*	注册读事件(OP_READ)到selector上


