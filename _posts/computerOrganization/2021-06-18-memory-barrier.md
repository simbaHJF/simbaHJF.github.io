---
layout:     post
title:      "内存屏障与原子性,可见性,有序性"
date:       2021-06-11 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 计算机组成原理

---



## 导航
[一. synchronized内存屏障](#jump1)
<br>
[二. volatile内存屏障](#jump2)
<br>





<br><br>
## <span id="jump1">一. synchronized内存屏障</span>

[![2Qr1Gq.jpg](https://z3.ax1x.com/2021/06/02/2Qr1Gq.jpg)](https://imgtu.com/i/2Qr1Gq)

<br>
**<font size="4">原子性</font>** <br>

synchronized原子性是通过加锁保障的


<br>
**<font size="4">可见性</font>** <br>

Load屏障,加锁会触发底层CPU高速缓存架构里的refresh操作,从主内存读取最新值<br>

Store屏障,释放锁会触发底层CPU高速缓存架构里的flush操作,把数据修改刷回主内存,确保对其他CPU核心的可见性<br>


<br>
**<font size="4">有序性</font>** <br>

Acquire屏障,禁止读操作和前面的读写操作发生指令重排序.<br>

Release屏障,禁止写操作和前面的读写操作发生指令重排序.<br>

这里需要注意,Acquire屏障和Release屏障可保证synchronized同步块内的代码与外部代码不会发生指令重排序,但在synchronized内部,也即Acquire屏障和Release屏障之间的指令,是可以重排序的.<br>



<br><br>
## <span id="jump2">二. volatile内存屏障</span>

```
volatile boolean isRunning = true;

线程1:

Release屏障
isRunning = false;
Store屏障


线程2:

Load屏障
while(isRunning){
    Acquire屏障
    //代码逻辑
}
```

<br>
**<font size="4">原子性</font>** <br>

单变量赋值或读取,本身就是原子性的指令.但是注意,volatile变量的++操作,可不是原子的,底层它其实是由多个指令组合的.<br>


<br>
**<font size="4">可见性和有序性</font>** <br>

对volatile修饰的变量,执行写操作的话,会生成一条lock前缀指令给CPU,而底层CPU在对该指令的实现,则是通过内存屏障保证的.<br>

在volatile变量写操作的前面会加入一个Release屏障,然后再之后会加入一个Store屏障,这样就可以保证volatile写跟Release屏障之前的任何读写操作都不会指令重排,然后Store屏障保证了,写完数据之后,立马会执行flush处理器缓存的操作<br>

在volatile变量读操作的前面会加入一个Load屏障,这样就可以保证对这个变量的读取时,如果被别的处理器修改过了,必须得从其他处理器的高速缓存(或主内存)中加载到自己的本地高速缓存里,也就是refresh,保证读到的是最新数据;在之后会加入一个Acquire屏障,禁止volatile读操作之后的任何读写操作跟volatile读指令重排序.<br>

