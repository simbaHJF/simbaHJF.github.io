---
layout:     post
title:      "mysql基础架构"
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


