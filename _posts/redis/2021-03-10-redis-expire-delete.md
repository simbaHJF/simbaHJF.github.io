---
layout:     post
title:      "redis 过期键删除策略"
date:       2021-03-10 10:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - redis

---



## 导航
[一. redis 过期键的存储](#jump1)
<br>
[二. redis 过期键删除策略](#jump2)
<br>
[三. AOF、RDB和复制功能对过期键的处理](#jump3)
<br>
[四. 小结](#jump4)









<br><br>
## <span id="jump1">一. redis 过期键存储</span>

redis通过过期字典保存数据库中所有键的过期时间:
* 过期字典的键是一个指针,这个指针指向键空间中的某个键对象(也即是某个数据库键)
* 过期字典的值保存了键所指向的数据库键的过期时间 ----- 一个毫秒精度的UNIX时间戳

下图是一个带有过期字典的数据库例子,在这个例子中,键空间保存了数据库中的所有键值对,而过期字典保存了数据库键的过期时间

[![6Gl3WD.png](https://s3.ax1x.com/2021/03/10/6Gl3WD.png)](https://imgtu.com/i/6Gl3WD)

当通过命令新增一个键的过期时间时
```
PEXPIREAT message 1391234400000
```

此时数据库中的数据如下所示:
[![6GGBHs.png](https://s3.ax1x.com/2021/03/10/6GGBHs.png)](https://imgtu.com/i/6GGBHs)


再通过命令移除一个键的过期时间:
```
PERSIST message
```

此时数据库中的数据如下所示:
[![6GJhz8.png](https://s3.ax1x.com/2021/03/10/6GJhz8.png)](https://imgtu.com/i/6GJhz8)



<br><br>
## <span id="jump2">二. redis 过期键删除策略</span>

<br>
**<font size="4">惰性删除 ----- 被动删除</font>** <br>

程序只会在取出键时才对键进行过期检查,这可以保证删除过期键的操作只会在非做不可的情况下进行,并且删除的目标仅限于当前处理的键,这个策略不会在删除其他无关的过期键上花费任何CPU时间.<br>

惰性删除的缺点是,它对内存时不友好的:如果一个键已经过期,而这个键又仍然保留在数据库中,那么只要这个过期键不被删除,它所占用的内存就不会释放.<br>


<br>
**<font size="4">定期删除 ----- 主动删除</font>** <br>

定期删除是指,每隔一段时间执行一次删除过期键操作,并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响.通过定期删除策略,能有效地减少因为过期键而带来的内存浪费.<br>


<br>
**<font size="4">惰性删除策略的实现</font>** <br>

所有读写数据库的Redis命令在执行之前都会调用 expireIfNeeded 函数对输入进行检查:
* 如果输入键已经过期,那么 expireIfNeeded 函数将输入键从数据库中删除
* 如果输入键未过期,那么 expireIfNeeded 函数不做动作

命令调用 expireIfNeeded 函数的过程如下图所示:
[![6GW7gU.png](https://s3.ax1x.com/2021/03/10/6GW7gU.png)](https://imgtu.com/i/6GW7gU)

举个例子,如下图调用 GET 命令的执行过程,再次过程中,命令需要判断键是否存在以及键是否过期,然后根据判断来执行合适的动作.
[![6GfrZ9.png](https://s3.ax1x.com/2021/03/10/6GfrZ9.png)](https://imgtu.com/i/6GfrZ9)


<br>
**<font size="4">定期删除策略的实现</font>** <br>

定期删除策略由函数 activeExpireCycle 实现, 其工作模式可以总结如下:
* 函数每次运行时,都从一定数量的数据库中去除一定数量的随机键进行检查,并删除其中的过期键
* 全局变量current_db会记录当前activeExpireCycle函数检查的进度,并在下一次activeExpireCycle函数调用时,接着上一次的进度进行处理.例如,若当前activeExpireCycle函数在遍历10号数据库时返回了,那么下次activeExpireCycle函数执行时,将从11号数据库开始查找并删除过期键.
* 随着activeExpireCyle函数的不断执行,服务器中的所有数据库都会被检查一遍,这时函数将current_db变量重置为0,然后再次开始新一轮的检查.



<br><br>
## <span id="jump3">三. AOF、RDB和复制功能对过期键的处理</span>

<br>
**<font size="4">生成RDB文件</font>** <br>

在执行SAVE命令或者BGSAVE命令创建一个新的RDB文件时,程序会对数据库中的键进行检查,已过期的键不会被保存到新创建的RDB文件中.<br>

<br>
**<font size="4">载入RDB文件</font>** <br>

在启动Redis服务器时,如果服务器开启了RDB功能,那么服务器将对RDB文件进行载入:
* 如果服务器以主服务器模式运行,那么在载入RDB文件时,程序会对文件中保存的键进行检查,未过期的键会被载入到数据库中,而过期的键则会被忽略,所以过期键对载入RDB文件的主服务器不会造成影响.
* 如果服务器以从服务器模式运行,那么在载入RDB文件时,文件中保存的所有键,不论是否过期,都会被载入到数据库中.不过,因为主从服务器在进行数据同步的时候,从服务器的数据库就会被清空,所以一般来讲,过期键对载入RDB文件的从服务器也不会造成影响


<br>
**<font size="4">AOF文件写入</font>** <br>

当服务器以AOF持久化模式运行时,如果数据库中的某个键已经过期,但它还没有被惰性删除或者定期删除,那么AOF文件不会因为这个过期键儿产生任何影响(因为没有对这个键的增量操作,因此没有相关AOF记录).<br>

当过期键被惰性删除或者被定期删除之后,程序会向AOF文件追加(append)一条DEL命令,来显示地记录该键已被删除.<br>

举个例子,如果客户端使用 GET message 命令,试图访问过期的 message 键,那么服务器将执行以下三个动作:
* 从数据库中删除 message 键
* 追加一条 DEL message 命令到AOF文件
* 向执行GET命令的客户端返回空回复


<br>
**<font size="4">AOF重写</font>** <br>

在执行AOF重写的过程中,程序会对数据库中的键进行检查,已过期的键不会被保存到重写后的AOF文件中.<br>


<br>
**<font size="4">复制</font>** <br>

当服务器运行在复制模式下时,从服务器的过期键删除动作由主服务器控制:
* 主服务器在删除一个过期键之后,会显示地向所有从服务器发送一个DEL命令,告知从服务器删除这个过期键
* 从服务器在执行客户端发送的读命令时,即使碰到过期键也不会将过期键删除,而是继续向处理未过期键一样来处理过期键
* 从服务器只有在接到主服务器发来的DEL命令之后,才会删除过期键

通过由主服务器来控制从服务器统一地删除过期键,可以保证主从服务器数据的一致性,也正是因为这个原因,当一个过期键仍然存在于主服务器的数据库时,这个过期键在从服务器里的复制品也会继续存在.<br>

举例来讲,有一对主从服务器,他们的数据库中都保存着同样的三个键message、xxx和yyy,其中message为过期键.
[![6GHUu6.png](https://s3.ax1x.com/2021/03/10/6GHUu6.png)](https://imgtu.com/i/6GHUu6)

如果这时有客户端向从服务器发送命令 GET message ,那么从服务器将发现message键已经过期,但从服务器并不会删除message键,而是继续将message键的值返回给客户端,就好像message键并没有过期一样.
[![6GHoCj.png](https://s3.ax1x.com/2021/03/10/6GHoCj.png)](https://imgtu.com/i/6GHoCj)

假如此后有客户端向主服务器发送命令 GET message ,那么主服务器将发现键message已经过期,主服务器会删除message键,向客户端返回空回复,并向从服务器发送 DEL message 命令.
[![6GbKxI.png](https://s3.ax1x.com/2021/03/10/6GbKxI.png)](https://imgtu.com/i/6GbKxI)

从服务器在收到主服务器发来的 DEL message 命令之后,也会从数据库中删除message键,在这之后,主从服务器都不再保存过期键message了.
[![6GLA4e.png](https://s3.ax1x.com/2021/03/10/6GLA4e.png)](https://imgtu.com/i/6GLA4e)



<br><br>
## <span id="jump4">四. 小结</span>

* Redis服务器的所有数据库都保存在redisServer.db数组中,而数据库的数量则由 redisServer.dbnum 属性保存
* 客户端通过修改目标数据库指针,让它指向redisServer.db数组中的不同元素来切换不同的数据库
* 数据库主要由dict和expires两个字典构成,其中dict字典负责保存键值对,而expires字典则负责保存键的过期时间
* 因为数据库由字典构成,所以对数据库的操作都是建立在字典操作之上的
* 数据库的键总是一个字符串对象,而值则可以是任意一种Redis对象类型,包括字符串对象、哈希表对象、集合对象、列表对象和有序集合,分别对应字符串键、哈希表键、集合键、列表建和有序集合键
* expires字典的键指向数据库中的某个键,而值则记录了数据库键的过期时间,过期时间是一个以毫秒为单位的UNIX时间戳
* Redis使用惰性删除和定期删除两种策略来删除过期的键:惰性删除策略只在碰到过期键时才进行删除操作,定期删除策略则每隔一段时间主动查找并删除过期键
* 执行SAVE命令或者BGSAVE命令所产生的的新RDB文件不会包含已过期的键
* 当一个过期键被删除之后,服务器会追加一条DEL命令到现有AOF文件的末尾,显示地删除过期键
* 从服务器即使发现过期键也不会自作主张删除它,而是等待主节点发来DEL命令,这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性