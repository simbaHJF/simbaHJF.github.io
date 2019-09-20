---
layout:     post
title:      "mysql事务"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---

> 极客时间--丁奇--mysql实战 学习笔记

##	事务
事务特性:ACID(Atomicity,Consistency,Isolation,Durability)

隔离级别与隔离性:

隔离级别 | 隔离性 |
-|-|
读未提交  |  脏读  |
读已提交  |  不可重复读  |
可重复读  |  幻读  |
串行化  |  -  |
