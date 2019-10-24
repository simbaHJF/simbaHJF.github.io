---
layout:     post
title:      "mysql -- 主备"
date:       2019-10-18 11:20:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---

> 极客时间--丁奇--mysql实战 学习笔记

##	MySQL主备的基本原理

基本的主备切换流程如下图所示:
![KV5rFS.png](https://s2.ax1x.com/2019/10/18/KV5rFS.png)


一条update语句在节点A执行,然后同步到节点B,节点A到B这条线的内部流程如下图:
![KVTyWD.png](https://s2.ax1x.com/2019/10/18/KVTyWD.png)

备库B跟主库A之间维持一个长连接.主库A内部有一个线程,专门用于服务备库B的这个长连接.一个事务日志同步的完整过程是这样的:

1.	在备库B上通过change master命令,设置主库A的IP,端口,用户名,密码以及要从哪个位置开始请求binlog,这个位置包含文件名和日志偏移量.
2.	在备库B上执行start slave命令,这时候备库会启动两个线程,就是图中的io_thread和sql_thread.其中io_thread
负责与主库建立连接.
3.	主库A交验完用户名,密码后,开始按照备库B传过来的位置,从本地读取binlog,发给B
4.	备库B拿到binlog后,写到本地文件,称为中转日志(relay log).
5.	sql_thread读取中转日志,解析出日志里的命令,并执行.


## 循环复制问题

实际生产上使用比较多的是双M结构,也就是下图所示结构:
![KZUGqI.png](https://s2.ax1x.com/2019/10/18/KZUGqI.png)

对比双M结构和M-S结构,其实区别只是多了一条线,即:节点A和B之间总是互为主备关系.这样在切换的时候就不用再修改主备关系.

但是双M结构还有一个问题需要解决.

业务逻辑在节点A上更新了一条语句,然后再把生成的binlog发给节点B,节点B执行完这条更新语句后也会生成binlog.(建议把log_slave_updates设置为on,表示备库执行relay log生成binlog).

那么,如果节点A同时是节点B的备库,相当于又把节点B新生成的binlog拿过来执行了一次,然后节点A和B之间,会不断的循环执行这个更新语句,也就是循环复制了.这个问题要怎么解决呢?

如上图所示,MySQL在binlog中记录了这个命令第一次执行时所在实例的server id.因此,我们可以用下面的逻辑,来解决两个节点间的循环复制问题:
1.	规定两个库的server id必须不同,如果相同,则他们之间不能设定为主备关系;
2.	一个备库接到binlog并在重放的过程中,生成与原binlog的server id相同的binlog;
3.	每个库在收到从自己的主库发过来的日之后,先判断server id,如果跟自己的相同,表示这个日志是自己生成的,就直接丢弃这个日志.

按照这个逻辑,如果我们设置了双M结构,日志的执行流就会变成这样:
1.	从节点A更新的事务,binlog里面记的都是A的server id;
2.	传到节点B执行一次后,节点B生成的binlog的server id也是A的server id;
3.	在传回给节点A,A判断到这个server id与自己的相同,就不会再处理这个日志.所以死循环就在这里断掉了.

