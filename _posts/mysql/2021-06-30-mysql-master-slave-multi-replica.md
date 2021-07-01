---
layout:     post
title:      "mysql -- 主从多线程复制"
date:       2021-06-30 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---







## 导航
[一. 多线程复制](#jump1)
<br>
[二. 基于DATABASE的多线程复制](#jump2)
<br>
[三. 基于LOGICAL_CLOCK的多线程复制](#jump3)
<br>
[四. 多线程复制模式下的事务执行顺序](#jump4)
<br>








<br><br>
## <span id="jump1">一. 多线程复制</span>

多线程复制MTS(Mult-Threaded Slave Applier)指使用多个线程来并发应用二进制日志.<br>

* 在 MySQL 5.6 版本中,多线程复制基于schema来实现,将多个数据库下的事务按照数据库拆分到多个线程上执行,保证数据库级别的事务一致性.
* 在 MySQL 5.7 版本后,多线程复制基于主库上并发信息来实现,主库上并发提交的事务不存在事务冲突,那么在从库上拆分到多个线程并发执行,可以保证实例级别的事务一致性. MySQL 5.7 中通过参数配置可以兼容5.6版本中的基于DATABASE多线程复制模式
    > slave_parallel_type: <br>
        DATABASE : 基于库的并行复制方式 <br>
        LOGICAL_CLOCK：基于组提交的并行复制方式 <br>



<br><br>
## <span id="jump2">二. 基于DATABASE的多线程复制</span>

在MySQL 5.6中引入该特性,如果主库上存在多个数据库,每个数据库的事务相互独立于其他数据库,因此只需要保证数据库内部的事务运行顺序和主库上的运行顺序一致,就可以保证主库和从库上的数据相同.<br>

[![R0qsC8.png](https://z3.ax1x.com/2021/06/30/R0qsC8.png)](https://imgtu.com/i/R0qsC8)


在MySQL中开启并行复制功能,SQL线程会变成coordinator线程,coordinator线程会对二进制日志的event进行判断:
* 如果判断事件可以被并行执行,那么选择相应worker线程应用BINLOG事件
* 如果判断事件不可以被并行执行,如DDL操作或跨schema事务,则等待所有worker线程执行完成后,再执行该BINLOG事件

coordinator线程不仅分发BINLOG事件,也可以执行BINLOG事件.<br>

当实例上数据库数量较少或应用主要对某个数据库进行读写,并行复制的性能可能会比单线程复制更差.<br>

对于跨数据库的事务或跨数据库的外键,都会导致无法多线程并行执行.<br>



<br><br>
## <span id="jump3">三. 基于LOGICAL_CLOCK的多线程复制</span>

在 MySQL 5.7 版本中引入,在主库上的某个时间点上,所有完成excution处于prepare阶段的事务都处于一个"相同的数据库版本"上,这些事务之间不存在阻塞或者依赖,因此可以赋予一个相同的时间戳;拥有相同时间戳的事务可以在从库上并行执行并且不会导致相互等待.如果事务间存在依赖,那么被阻塞的事务肯定处于Execution状态而不会进入Prepare状态<br>

[![R0OXh6.png](https://z3.ax1x.com/2021/06/30/R0OXh6.png)](https://imgtu.com/i/R0OXh6)

如上图中三个事务:
* T1事务和T2事务的Commit阶段有重合部分,T2事务和T3事务的Commit阶段有重合部分,因此T1和T2可以在从库上并发执行,T2和T3可以在从库上并发执行.
* T1事务和T3事务的Commit阶段没有有重合部分,无法判断T3事务是否依赖于T1事务,因此T1和T3不能在从库上并发执行

<font color="red">Transactions with overlapping commit window can be executed in parallel;</font>

在 MySQL 5.7 版本的二进制日志中增加了last_committed和sequence_number,sequence_number表示当前语句所使用的编号,使用last_committed表示当前语句提交时的上一次组提交事务中最大的sequence_number.相同last_committed的事件可以并行执行,无需考虑事件中的sequence_number.<br>

```
#150520 14:23:11 server id 88 end_log_pos 259 CRC32 0x4ead9ad6 GTID last_committed=0 sequence_number=1
#150520 14:23:11 server id 88 end_log_pos 1483 CRC32 0xdf94bc85 GTID last_committed=0 sequence_number=2
#150520 14:23:11 server id 88 end_log_pos 2708 CRC32 0x0914697b GTID last_committed=0 sequence_number=3
#150520 14:23:11 server id 88 end_log_pos 3934 CRC32 0xd9cb4a43 GTID last_committed=0 sequence_number=4
#150520 14:23:11 server id 88 end_log_pos 5159 CRC32 0x06a6f531 GTID last_committed=0 sequence_number=5
#150520 14:23:11 server id 88 end_log_pos 6386 CRC32 0xd6cae930 GTID last_committed=0 sequence_number=6
#150520 14:23:11 server id 88 end_log_pos 7610 CRC32 0xa1ea531c GTID last_committed=6 sequence_number=7
#150520 14:23:11 server id 88 end_log_pos 8834 CRC32 0x96864e6b GTID last_committed=6 sequence_number=8
#150520 14:23:11 server id 88 end_log_pos 10057 CRC32 0x2de1ae55 GTID last_committed=6 sequence_number=9
#150520 14:23:11 server id 88 end_log_pos 11280 CRC32 0x5eb13091 GTID last_committed=6 sequence_number=10
#150520 14:23:11 server id 88 end_log_pos 12504 CRC32 0x16721011 GTID last_committed=6 sequence_number=11
#150520 14:23:11 server id 88 end_log_pos 13727 CRC32 0xe2210ab6 GTID last_committed=6 sequence_number=12
#150520 14:23:11 server id 88 end_log_pos 14952 CRC32 0xf41181d3 GTID last_committed=12 sequence_number=13
```

相同last_committed被认为是同一组,在从库端相同last_committed的事务可以由sql线程交给worker线程并发执行,类似如下图:

[url=https://imgtu.com/i/RBtsBV][img]https://z3.ax1x.com/2021/06/30/RBtsBV.jpg[/img][/url]


而MySQL主库是如何做到将这些事务分组的呢?这涉及到MySQL的提交方式:ordered_committed,原理图如下

[![RB0BjJ.png](https://z3.ax1x.com/2021/06/30/RB0BjJ.png)](https://imgtu.com/i/RB0BjJ)

大致实现原理就是:excecution阶段可以并行执行,binlog flush的时候,借助队列按顺序进行




<br><br>
## <span id="jump4">四. 多线程复制模式下的事务执行顺序</span>

MySQL通过参数slave_preserve_commit_order可以控制Slave上的binlog提交顺序和Master上的binlog的提交顺序一样,保证GTID的顺序.该参数只能用于开启了logical clock并且启用了binlog的复制.即对于多线程复制,该参数用来保障事务在slave上执行的顺序与relay log中的顺序严格一致.开启该参数可能会有一点的消耗,因为会让slave的binlog提交产生等待.<br>

比如两个事务依次操作了2个DB:A和B,尽管事务A、B分别被worker X、Y线程接收,但是因为线程调度的问题,有可能导致A的执行时机落后于B.如果经常是"跨DB"操作,那么可以考虑使用此参数限定顺序.当此参数开启时,要求任何worker线程执行事务时,只有当前事务中之前的所有事务都执行后(被其他worker线程执行),才能执行和提交.(每个事务中,都记录了当前GTID的privious GTID,只有privious GTID被提交后,当前GTID事务才能提交).

建议在生产环境开启该参数。