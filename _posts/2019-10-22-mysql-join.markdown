---
layout:     post
title:      "mysql -- join"
date:       2019-10-22 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---

> 极客时间--丁奇--mysql实战 学习笔记

##	join

在实际生产中,关于join语句的使用问题,一般会集中在以下两类:
1.	DBA不让使用join,使用join有什么问题呢?
2.	如果有两个大小不同的表做join,应该用哪个表做驱动表?


###	join是怎么执行的

为了便于量化分析,先创建两个表t1和t2:
```

CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

这两个表都有一个主键索引id和一个索引a,字段b上无索引.存储过程idata()往表t2里插入了1000行数据,在表t1里插入的是100行数据.


####	Index Nested-Loop Join

有如下语句:
```
select * from t1 straight_join t2 on (t1.a=t2.a);
```

使用straight_join时为了让MySQL使用固定的连接方式执行查询而不让优化器改变连接方式.在这个语句里,t1是驱动表,t2是被驱动表.这条语句的explain结果如下:
![K82skd.png](https://s2.ax1x.com/2019/10/22/K82skd.png)

这条语句里,被驱动表t2的字段a上有索引,join过程用上了这个索引,因此这个语句的执行流程是这样的:
1.	从表t1中读入一行数据R
2.	从数据行R中,去除a字段到表t2里去查找
3.	取出表t2中满足条件的行,跟R组成一行,作为结果集的一部分
4.	重复执行步骤1到3,直到表t1的末尾循环结束

这个过程是先遍历表t1,然后根据从表t1中取出的每行数据中的a值,去表t2中查找满足条件的记录.它对应的流程图如下:
![K8Ivxs.jpg](https://s2.ax1x.com/2019/10/22/K8Ivxs.jpg)

在这个流程里:
1.	对驱动表t1做了全表扫描,这个过程需要扫描100行;
2.	对于每一行R,根据a字段去表t2查找,走的时候树搜索过程.由于我们构造的数据都是一一对应的,因此每次的搜索过程都只扫描一行,也是总共扫描100行.
3.	所以整个执行流程总扫描行数是200

现在来看一下文章开头的两个问题.

第一个问题:能不能使用join

假设不使用,那么就只能单表查询.上面这条语句的实现逻辑是:
1.	执行select * from t1,查出表t1的所有数据,这里有100行
2.	循环遍历这100行数据:
	*	从每一行R取出字段a的值$R.a
	*	执行select * from t2 where a=$R.a
	*	把返回的结果和R构成结果集的一行.

在这个查询过程中,也是扫描了200行,但是总共执行了101条语句,比直接join多了100次交互.除此之外,客户端还要自己拼接SQL语句和结果.显然,这么做还不如直接join好.

第二个问题:怎么选择驱动表?

在这个join语句执行过程中,驱动表是走全表扫描,而被驱动表是走树搜索.假设被驱动表的行数是M.每次在被驱动表查一行数据,要先搜索索引a,再搜索主键索引.每次搜索一棵树近似复杂度是以2为底的M的对数,记为logM,所以在被驱动表上查一行的时间复杂度是2*logM(普通索引回表主键索引).

假设驱动表的行数是N,执行过程就要扫描驱动表N行,然后对于每一行,到被驱动表上匹配一次,因此整个执行过程,近似复杂度是  N + N * 2 * logM.

显然,N对扫描行数的影响更大,因此应该让小表来做驱动表.
**假如N扩大1000倍的话,扫描行数就会扩大1000倍;而M扩大1000倍的话,扫描行数扩大不到10倍**

通过上面分析得到两个结论:
1.	使用join语句,性能比强行拆成多个单表执行SQL语句的性能要好
2.	如果使用join语句的话,需要让小表做驱动表

<font color="red">但是,需要注意,这个结论的前提是"可以使用被驱动表的索引"</font>

接下来,再来看看被驱动表用不上索引的情况.

####	Simple Nested-Loop Join

现在,我们把SQL语句改成这样:
```
select * from t1 straight_join t2 on (t1.a=t2.b);
``` 

