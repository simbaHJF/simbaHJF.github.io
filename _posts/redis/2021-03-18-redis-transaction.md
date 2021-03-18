---
layout:     post
title:      "redis transaction"
date:       2021-03-18 12:40:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - redis

---



## 导航
[一. 事务的实现](#jump1)
<br>
[二. WATCH命令](#jump2)
<br>
[三. Redis事务的ACID性质](#jump3)












<br><br>
## <span id="jump1">一. 事务的实现</span>

一个事务从开始到结束通常会经历以下三个阶段:
* 事务开始
* 命令入队
* 事务执行


<br>
**<font size="4">事务开始</font>** <br>

MULTI命令的执行标志着事务的开始,MULTI命令可以将执行该命令的客户端从非事务状态切换至事务状态


<br>
**<font size="4">命令入队</font>** <br>

当一个客户端处于非事务状态时,这个客户端发送的命令会立即被服务器执行
[![6gTdrq.png](https://s3.ax1x.com/2021/03/18/6gTdrq.png)](https://imgtu.com/i/6gTdrq)

与此不同的是,当一个客户端切换到事务状态之后,服务器会根据这个客户端发来的不同命令执行不同的操作:
* 如果客户端发送的命令为 EXEC、DISCARD、WATCH、MULTI 四个命令的其中一个,那么服务器立即执行这个命令
* 与此相反,如果客户端发送的命令是除上述4个命令的其他命令,那么服务器并不立即执行这个命令,而是将这个命令放入一个事务队列里面,然后向客户端返回QUEUED回复

[![6gTOsI.png](https://s3.ax1x.com/2021/03/18/6gTOsI.png)](https://imgtu.com/i/6gTOsI)


<br>
**<font size="4">事务队列</font>** <br>

事务队列以先进先出(FIFO)的方式保存入队的命令,例如客户端执行以下命令的结果:
[![6g7gk8.png](https://s3.ax1x.com/2021/03/18/6g7gk8.png)](https://imgtu.com/i/6g7gk8)


<br>
**<font size="4">执行事务</font>** <br>

当一个处于事务状态的客户端向服务器发送EXEC命令时,这个EXEC命令将立即被服务器执行.服务器会遍历这个客户端的事务队列,执行队列中保存的所有命令,最后将执行命令所得的结果全部返回给客户端<br>



<br><br>
## <span id="jump2">二. WATCH命令</span>

WATCH命令是一个乐观锁,它可以在EXEC命令执行之前,监视任意数量的数据库键,并在EXEC命令执行时,检查被监视的键是否至少有一个已经被修改过了,如果是的话,服务器将拒绝执行事务,并向客户端返回代表事务执行失败的空回复.<br>

以下是一个事务执行失败的例子:
[![6gH39g.png](https://s3.ax1x.com/2021/03/18/6gH39g.png)](https://imgtu.com/i/6gH39g)

下表展示了上面的例子是如何失败的
[![6gHNBq.png](https://s3.ax1x.com/2021/03/18/6gHNBq.png)](https://imgtu.com/i/6gHNBq)



<br><br>
## <span id="jump3">三. Redis事务的ACID性质</span>

在Redis中,事务总是具有原子性(Atomicity)、一致性(Consistency)和隔离性(Isolation),并且当Redis运行在某种特定的持久化模式下时,事务也具有耐久性(Durability)


<br>
**<font size="4">原子性</font>** <br>

对于Redis的事务功能来说,事务队列中的命令要么就全部都执行,要么就一个都不执行.<br>

以下是一个成功执行的事务示例,事务中的所有命令都会被执行:
```
redis > MULTI
OK

redis > SET msg "hello"
QUEUED

redis > EXEC
1) OK
2) "hello"
```

下面是一个执行失败的事务示例,这个事务因为命令入队出错而被服务器拒绝执行
[![6gb7JU.png](https://s3.ax1x.com/2021/03/18/6gb7JU.png)](https://imgtu.com/i/6gb7JU)

<font color="red">Redis事务中,命令只要入队成功了,就会全部执行,哪怕执行过程中出现了错误,后续命令也会执行下去(这点也就是常说的Redis不支持事务回滚),除非在命令入队阶段就判断命令错误,那么就会全部不执行</font> <br>

如下例子,即使RPUSH命令在执行期间出现了错误,事务的后续命令也会继续执行下去,直到将事务队列中的所有命令都执行完毕为止
[![6gqpFK.png](https://s3.ax1x.com/2021/03/18/6gqpFK.png)](https://imgtu.com/i/6gqpFK)


<font color="red">Redis的作者在事务功能的文档中解释说,不支持事务回滚是因为这种复杂的功能和Redis追求简单高效的设计主旨不相符,并且他认为,Redis事务的执行时错误通常都是编程错误产生的,因此没有必要为Redis开发事务回滚功能</font>


<br>
**<font size="4">一致性</font>** <br>

* 入队错误 ----- 如果一个事务在入队命令的过程中,出现了命令不存在,或者命令的格式不正确等情况,那么Redis将拒绝执行这个事务
[![6gqfpD.png](https://s3.ax1x.com/2021/03/18/6gqfpD.png)](https://imgtu.com/i/6gqfpD)

* 执行错误 ----- 对数据库键执行了错误类型的操作是事务执行期间最常见的错误之一
	* 执行过程中发生的错误都是一些不能在入队时被服务器发现的错误,这些错误只会在命令实际执行时被触发
	* 即使在事务的执行过程中发生了错误,服务器也不会中断事务的执行,它会继续执行事务中余下的命令,并且一致性的命令(包括执行命令所产生的的结果)不会被出错的命令影响
[![6gLE3F.png](https://s3.ax1x.com/2021/03/18/6gLE3F.png)](https://imgtu.com/i/6gLE3F)

* 服务器停机


<br>
**<font size="4">隔离性</font>** <br>

Redis单线程的方式来执行事务(以及事务队列中的命令),并且服务器保证,在执行事务期间不会对事务进行中的,因此Redis事务总是以串行的方式运行的,总是具有隔离性.<br>


<br>
**<font size="4">持久性</font>** <br>

与同步刷盘策略有关