---
layout:     post
title:      "OOM与CPU 100% 排查方法论"
date:       2021-03-22 10:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JVM

---






## 导航
[一. OOM排查](#jump1)
<br>
[二. CPU 100%排查](#jump2)
<br>










<br><br>
## <span id="jump1">一. OOM排查</span>

前提,需要在系统启动参数中增加参数设置,在发送OOM造成JVM退出时,dump内存快照<br>
``
-XX:+HeapDumpOnOutOfMemoryError
``

排查步骤流程:
1. 登录机器,查看错误日志
2. 查看具体是StackOverFlowError还是OutOfMemoryError
	* 对于StackOverFlowError,查看是哪个方法哪行造成溢出,排查是否有无限递归或错误超长方法调用栈
	* 对于OutOfMemoryError,将溢出时dump的内存快照拉到本地,用MAT等工具进行分析,查看是哪类对象较多造成的溢出



<br><br>
## <span id="jump2">二. CPU 100%排查</span>

1. ``top -c``,查看cpu使用率最高的进程的pid
2. ``top -H -p pid``,查看进程pid下线程的cpu使用率,并找出cpu使用率最高的线程
3. ``printf "%x\n" 线程pid``,线程pid转为16进制
4. ``jstack 进程pid| grep '线程pid的16进制'``,查看是哪个线程cpu使用率高,是否有死循环等