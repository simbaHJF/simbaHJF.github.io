---
layout:     post
title:      "硬件层级下再看CPU高速缓存架构及MESI协议"
date:       2021-06-11 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 计算机组成原理

---





## 导航
[一. 硬件层架下的CPU高速缓存架构及MESI协议](#jump1)
<br>
[二. CPU高速缓存架构下使用写缓冲器和无效队列优化性能](#jump2)
<br>
[三. 硬件层面MESI协议引发的有序性和可见性的问题](#jump3)
<br>
[四. 内存屏障在硬件层面的实现原理以及如何解决有序性和可见性问题](#jump4)
<br>
[五. 注意注意](#jump5)
<br>
[六. CAS是如何基于MESI协议在底层硬件层面实现加锁的](#jump6)









<br><br>
## <span id="jump1">一. 硬件层架下的CPU高速缓存架构及MESI协议</span>

[![2fSvb8.png](https://z3.ax1x.com/2021/06/11/2fSvb8.png)](https://imgtu.com/i/2fSvb8)



<br><br>
## <span id="jump2">二. CPU高速缓存架构下使用写缓冲器和无效队列优化性能</span>

在最基础的CPU高速缓存架构及MESI协议下,读写数据的性能效率较低,因此各CPU核心又引入了写缓冲器和无效队列,如上图所示.在加入了写缓冲器和无效队列后,读写数据的流程如下:
1. CPU-0想要写入数据,直接写入写缓冲器中,然后向总线发送invalidate消息,这时就认为写操作完成,不会阻塞,立即返回,可以紧接着执行后续操作了.当然这时候还未更新高速缓存中的数据
2. CPU-1嗅探到invalidate消息后,直接将其写入无效队列,不会阻塞立即返回,然后直接返回invalidate ack消息.当然此时CPU-1也还未更新其高速缓存中的数据
3. CPU-0收到所有其他CPU的invalidate ack后,从写缓冲器取出那条写的数据,将高速缓存中对应cache entry的数据状态变为E,此时完成了加独占锁,然后进行数据更改写入,完成后数据状态变为M
4. CPU-1之后的某个时间会慢慢从无效队列里消费invalidate消息,过期本地高速缓存中cache entry的数据,状态变为I



<br><br>
## <span id="jump3">三. 硬件层面MESI协议引发的有序性和可见性的问题</span>

正是因为写缓冲器和无效队列的引入,导致了硬件层面MESI协议会产生有序性和可见性问题.<br>


<br>
**<font size="4">可见性</font>** <br>

写缓冲器和无效队列导致,写数据不一定立即写入自己的高速缓存(或者主存);读数据不一定能立即从高速缓存(或者主存)读到最新数据


<br>
**<font size="4">有序性</font>** <br>

* Store Load 重排序

```
int a = 0;
int c = 1;

线程1:
int a = 0;
int b = c;
```

CPU在运行时可能对a的赋值写入的Store操作,是先写入了写缓冲器中,此时高速缓存和主内存中都没有;然后对局部变量b的赋值是Load操作. <br>

这就导致以别的CPU核心中的线程来观察的话,对a的写入好像没有执行(因为是写入到写缓冲器中的),而对b的赋值先执行了,好像指令被重排序了.<br>


* Store Store 重排序

```
resource = loadResource();
loaded = true;
```

可能此时,CPU-0中的高速缓存中,对resource的cache entry是S状态,表示别的核心中也有该数据,此时写入时,是将其写入到写缓冲区中的;而对loaded的cache entry是M状态,说明CPU-0在之前已经取得了该数据的锁了,其他核心中的该数据都是过期的I,可以直接更新高速缓存.<br>

这时以外部其他CPU核心中的线程视角来观察时,好像对两个变量的写操作顺序发生了指令的重排序.<br>



<br><br>
## <span id="jump4">四. 内存屏障在硬件层面的实现原理以及如何解决有序性和可见性问题</span>

<br>
**<font size="5">可见性方面</font>** <br>

Store屏障 + Load屏障<br>

如果加了Store屏障之后,就会强制性要求对一个写操作必须阻塞等待到收到所有其他处理器返回的invalidate ack之后,对数据加锁,然后修改数据到高速缓存中,必须在写数据之后,强制执行flush操作.它的效果是,要求一个写操作必须刷到高速缓存(或者主内存),不能停留在写缓冲里<br>

如果加了Load屏障之后,在从高速缓存中读取数据的时候,如果发现无效队列里有一个invalidate消息,此时会立即强制根据那个invalidate消息把自己本地高速缓存中的数据置为失效I,然后就可以强制从其他处理器的高速缓存中(或者主存中)加载最新的值,这就是refresh操作.<br>


<br>
**<font size="5">有序性方面</font>** <br>

[![2fNiKs.png](https://z3.ax1x.com/2021/06/11/2fNiKs.png)](https://imgtu.com/i/2fNiKs)

内存屏障,Acquire屏障,Release屏障,但是都是有基础的StoreStore屏障,StoreLoad屏障所组成的,来避免指令的重排序.<br>

StoreStore屏障,会强制让写数据的操作全部按照顺序写入写缓冲器里,他不会让第一个写到写缓冲器里,然后第二个写直接修改高速缓存了.<br>

StoreLoad屏障,它会强制先将写缓冲器里的数据写入高速缓存中,接着读数据的时候强制清空无效队列,对立面的validate消息全部过期掉高速缓存中的条目,然后强制从主内存里重新加载数据.<br>



<br><br>
## <span id="jump5">五. 注意注意</span>

这里有一点要提出,不要把store指令和store屏障弄混了,指令时指令,屏障是屏障.都叫store,但不是一个东西.<br>



<br><br>
## <span id="jump6">六. CAS是如何基于MESI协议在底层硬件层面实现加锁的</span>

1. 基于MESI协议,发送invalidate消息,对数据加独占锁,状态变为 E
2. 数据查出来,进行比较
3. expectValue一样,则更新为newValue,状态变为M