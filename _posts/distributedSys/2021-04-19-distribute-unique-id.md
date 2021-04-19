---
layout:     post
title:      "分布式系统的唯一id生成方案"
date:       2021-04-19 10:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 分布式

---








## 导航
[一. 独立数据库自增id](#jump1)
<br>
[二. uuid](#jump2)
<br>
[三. 获取系统当前时间](#jump3)
<br>
[四. snowflake](#jump4)
<br>










<br><br>
## <span id="jump1">一. 独立数据库自增id</span>

这个方案就是说系统每次要生成一个id,都是往一个独立库的一个独立表里插入一条没什么业务含义的数据,然后获取一个数据库自增的一个id.以此作为分布式唯一id.<br>

优缺点:
* 优点: 简单方便
* 缺点: 单库生成自增id,高并发下,性能瓶颈显著.



<br><br>
## <span id="jump2">二. uuid</span>

UUID,你懂得.<br>

优缺点:
* 优点: 生成简单
* 缺点: 太长了,不适合作为数据库主键



<br><br>
## <span id="#jump3">三. 获取系统当前时间</span>

高并发下会有重复问题,可以拼接其他业务字段来进行唯一化.<br>



<br><br>
## <span id="jump4">四. snowflake</span>

Twitter开源的分布式id生成算法.<br>

其核心思想就是: 使用一个64bit的long型的数字作为全局唯一id.这64位中包括:
* 41bit作为毫秒数
* 10bit作为工作机器id
* 12bit作为序列号
* 1bit不用

序列号是为了避免高并发下,相同的工作机器id下,毫秒数相同的情况出现.<br>