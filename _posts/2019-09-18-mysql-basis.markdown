---
layout:     post
title:      "mysql基础"
date:       2019-09-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---

> 极客时间--丁奇--mysql实战 学习笔记


##	mysql的基础架构
大体来说,mysql可以分为server层和存储引擎层两部分.

server层包括连接器,查询缓存,分析器,优化器,执行器等,涵盖mysql的大多数核心服务功能,以及所有的内置函数(如日期,时间,数学和加密函数等),所有跨存储引擎的功能都在这一层实现,比如存储过程,触发器,视图等.

存储引擎层负责数据的存储和提取.其架构模式是插件式的,支持InnoDB,MyISAM,Memory等多个存储引擎.现在最常用的存储引擎是InnoDB,它从mysql 5.5.5版本开始成为了默认存储引擎.

mysql的逻辑架构图如下:
[![nHBXwt.png](https://s2.ax1x.com/2019/09/18/nHBXwt.png)](https://imgchr.com/i/nHBXwt)

不同的存储引擎公用一个server层,也就是从连接器到执行器的部分.

查询缓存往往弊大于利,大多数情况下建议不要使用查询缓存.mysql 8.0版本直接将查询缓存的整块功能删掉了.

##	sql查询语句执行顺序
![nH6qaR.png](https://s2.ax1x.com/2019/09/18/nH6qaR.png)

1.	from子句组装来自不同数据源的数据
2.	where子句基于指定的条件对记录进行筛选
3.	group by子句将数据划分为多个分组
4.	使用聚集函数进行计算
5.	使用having子句筛选分组
6.	计算所有的表达式
7.	select的字段
8.	使用order by对结果集进行排序.


##	redo log
redo log是InnoDB引擎特有的日志,InnoDB通过redo log来保证crash-safe.

redo log是物理日志,记录的是"在某个数据页上做了什么修改".

mysql进行update操作时,采用WAL(Write-Ahead Logging),即先写日志,再写磁盘.当有一条记录需要更新的时候,InnoDb引擎就会先把记录写到redo log里面,并更新内存,这个时候更新就算完成了.同时,InnoDB引擎会在适当的时候,将这个操作记录更新到磁盘里面,而这个更新往往是在系统比较闲的时候做.

InnoDB的redo log是固定大小的,比如可以配置为一组4个文件,每个文件的大小是1GB.从头开始写,写到末尾就又回到开头循环写.如下图所示:
![nHxQ0J.png](https://s2.ax1x.com/2019/09/18/nHxQ0J.png)
write pos是当前记录的位置,一边写以便后移,写到第3号文件末尾后就回到0号文件开头.checkpoint是当前要擦除的位置,也是往后推移并且循环的,擦除记录前要把记录更新到磁盘.


##	binlog
binlog是mysql的server层日志.最开始mysql没有InnoDB引擎,它的自带引擎是MyISAM,但是MyISAM没有crash-safe能力,binlog日志只能用于归档.

binlog是逻辑日志,记录的是这个语句的原始逻辑,比如"给ID=2这一行的c字段加1".

binlog是追加写入的,binlog文件写到一定大小后会切换到下一个,并不会覆盖以前的日志.

##	更新语句的执行过程
```
update T set c=c+1 where ID=2;
```

1.	执行器先找引擎取ID=2这一行.ID是主键,引擎直接用B+树搜索找到这一行.如果ID=2这一行所在的页本来就在内存中,就直接返回给执行器;否则,需要从磁盘读入内存,然后再返回.
2.	执行器拿到引擎给的行数据,把这个值加上1,比如原来是N,现在就是N+1,得到新的一行数据,再调用引擎接口写入这行新数据.
3.	引擎将这行新数据更新到内存中,同时将这个更新操作记录到redo log里面,此时redo log处于prepare状态.然后告知执行器执行完成了,随时可以提交事务.
4.	执行器生成这个操作的binlog,并把binlog写入磁盘.
5.	执行器调用引擎的提交事务接口,引擎把刚刚写入的redo log改成提交(commit)状态,更新完成.

下面是这个update语句的执行流程图,图中浅色框表示是在InnoDB内部执行的,深色框表示是在执行器中执行的:
![nbCiid.png](https://s2.ax1x.com/2019/09/19/nbCiid.png)

上述执行过程将redo log的写入拆成了两个步骤:prepare和commit,这就是"两阶段提交".


##	怎样让数据库恢复到半个月内任意一秒的状态
当需要恢复到指定的某一秒时,比如某天下午两点发现中午十二点有一次误删表,需要找回数据,那么可以这么做:
*	首先找到最近的一次全量备份,从这个备份恢复到临时库
*	然后,从备份的时间点开始,将备份的binlog依次取出来,重放到中午误删表之前的那个时刻.

