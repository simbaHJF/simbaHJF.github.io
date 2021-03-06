---
layout:     post
title:      "LINUX系统select函数原理"
date:       2019-06-10 22:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - linux

---


首先,先介绍一下select函数的调用流程:
1.  设置文件描述符

     指定监视范围
     
     设置超时时间
2.  调用select函数
3.  查看调用结果

具体函数格式为:
```
int select(int maxfdp,fd_set *readfds,fd_set *writefds,fd_set *errorfds,struct timeval *timeout)
```

下面结合select函数格式,参数和调用步骤进行具体说明

###  select函数的参数和返回值
#####  fd_set
struct fd_set可以理解为一个集合,这个集合中存放的是文件描述符(filedescriptor),即文件句柄,其结构可以用下面的图来描述:
![fd_set结构描述图](https://s2.ax1x.com/2019/06/12/V2x5YF.png)
图中的fd0,fd1,fd2......等表示文件描述符0,1,2......等(位置).如果该位值为1,则表示该文件描述符是监视对象;值为0表示该文件描述符不是监视对象.

*  fd_set* readfds:是指向fd_set结构的指针,这个集合中包含了一系列文件描述符,我们希望监视这些文件描述符的读变化,即是否可以从这些文件中读取数据了.如果这个集合中有一个文件可读,select就会返回一个大于0的值,表示有文件可读,如果没有可读的文件,则根据timeout参数再判断是否超时,若超出timeout的时间,select返回0,若发生错误返回负值.可以传入NULL值,表示不关心任何文件的读变化.
*  fd_set* writefds:是指向fd_set结构的指针,这个集合中包含了一系列文件描述符,我们希望监视这些文件描述符的写变化,即是否可以向这些文件 中写入数据了.如果这个集合中有一个文件可写,select就会返回一个大于0的值,表示有文件可写,如果没有可写的文件,则根据timeout参数再判断是否超时,若超出timeout的时间,select返回0,若发生错误返回负值.可以传入NULL值,表示不关心任何文件的写变化. 
*  fd_set * errorfds:同上面两个参数的意图,用来监视文件错误异常.


fd_set集合可以通过一些宏来操作，比如： 
*  FD_ZERO(fd_set *)
   清空集合 
*  FD_SET(int,  fd_set *)
   将一个给定的文件描述符加入集合之中
*  FD_CLR(int,  fd_set*)
   将一个给定的文件描述符从集合中删除
*  FD_ISSET(int,  fd_set*)
   检查集合中指定的文件描述符是否可以读写

下图说明上面几个宏的应用:
![](https://s2.ax1x.com/2019/06/12/VRmkFJ.png)


#####  timeval
struct timeval 是一个代表时间值的数据结构,其有两个成员,一个是秒数,另一个是微秒.其结构如下:
```
struct timeval
{
    long tv_sec;
    long tv_usec;
};
```
struct timeval* timeout：是select的超时时间,这个参数至关重要,它可以使select处于三种状态:
*  若将NULL以形参传入,即不传入时间结构,就是将select置于阻塞状态,一直等到监视文件描述符集合中某个文件描述符发生变化为止;
*  若将时间值设为0秒0微秒,就变成一个纯粹的非阻塞函数,不管文件描述符是否有变化,都立刻返回继续执行,文件无变化返回0,有变化返回一个正值;
*  timeout的值大于0,这就是等待的超时时间,即select在timeout时间内阻塞,超时时间之内有事件到来就返回了,否则在超时后不管怎样一定返回,返回值同上述.

#####  select 返回值： 
负值：select错误

正值：某些文件可读写或出错

0：等待超时，没有可读写或错误的文件

#####  int maxfdp
int maxfdp:是一个整数值,是指集合中所有希望监视的文件描述符的范围.这个参数单纯看说明不太好理解,结合后面select模型理解其含义就清楚了.
<br>
<br>
<br>
###  select模型
理解select模型的关键在于理解fd_set,为说明方便,取fd_set长度为1字节,fd_set中的每一bit可以对应一个文件描述符fd.则1字节长的fd_set最大可以对应8个fd.

（1）对于fd_set set;执行FD_ZERO(&set);则set用位表示是0000,0000.(这里假定左边为低位即fd0,右边为高位,对应前面的fd_set图)

（2）若fd＝5,执行FD_SET(fd,&set)后set变为0000,0100(index 5位置置为1)

（3）若再加入fd＝2,fd=1,则set变为0110,0100

（4）执行select(6,&set,0,0,NULL)阻塞等待  这里对第一个参数也就是maxfdp稍作说明,因为我们希望监视的文件描述符为在fd_set中最后的一个1对应的位置,而fd_set下标从0开始,所以maxfdp=6,即我们监视的范围到下标6对应的fd结束(不包括6,个人理解).

（5）若fd=1,fd=2上都发生可读事件,则select返回,此时set变为0110,0000.
注意:没有事件发生的fd=5被清空.这块很重要,一定要注意,举个例子,我们希望监视的文件描述符为fd1,fd2,fd3,select调用后fd1,fd3有事件就绪,fd2没有,看下图select函数调用前后的变化:
![](https://s2.ax1x.com/2019/06/12/VRmnOK.png)

<br>
<br>
基于上面的讨论，可以轻松得出select模型的特点：

（1)可监控的文件描述符个数取决与sizeof(fd_set)的值,也就是说select能监控的fd个数是有限的.据说可调,另有说虽然可调,但调整上限受限于编译内核时的变量值.

（2）将fd加入select监控集的同时,还要再使用一个数据结构array保存放到select监控集中的fd,一是用于在select返回后,array作为源数据和fd_set进行FD_ISSET判断.二是select返回后会把以前加入的但并无事件发生的fd清空,则每次开始 select前都要重新从array取得fd逐一加入(FD_ZERO最先),扫描array的同时取得fd最大值maxfd,用于select的第一个参数.

（3）可见select模型必须在select前循环array(加fd,取maxfd),select返回后循环array(FD_ISSET判断是否有事件发生).

上面总结的3点select模型特点借鉴自网络博客,加一点个人理解:上面的特点3就是影响select效率的问题所在,总需要遍历array进行fd_set的恢复;通过遍历array与select返回后的fd_set对比进行是否有事件发生.另外,对高并发而言,全部待监控连接可能是数以十万计的,返回的只是数百个活跃连接,而select的每次调用都需要传入全部待监控连接给内核,这是非常低效的.
