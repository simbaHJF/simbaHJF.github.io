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

由于表t2的字段b上没有索引,因此,如果再用前面图中所示流程的话,每次到t2区匹配的时候,就要做一次全表扫描.但是,这样算来,这个SQL请求就要扫描t2多达100次,总共扫描100*1000=10万行.这还只是两个小表,如果t1和t2都是10万行,就要扫描100亿行,这个算法看上去太"笨重"了.

当然,MySQL也没有使用这个Simple Nested-Loop Join算法,而是使用了另一个叫做"Block Nested-Loop Join"的算法,简称BNL.


####  Block Nested-Loop Join

这时候,被驱动表上没有可用的索引,算法的流程是:
1.  把表t1的数据读入线程内存join_buffer中,由于我们这个语句是select *,因此是把整个表t1放入了内存;
2.  扫描表t2,把表t2中的每一行取出来,跟join_buffer中的数据做对比,满足join条件的,作为结果集的一部分返回.

这个过程的流程图如下:
![KJ5H3Q.jpg](https://s2.ax1x.com/2019/10/23/KJ5H3Q.jpg)

相应地,这条SQL语句的explain结果如下所示:
![KJI8bt.png](https://s2.ax1x.com/2019/10/23/KJI8bt.png)

在这个过程中,对表t1和t2都做了一次全表扫描,因此总的扫描行数是1100.由于join_buffer是以无序数组的方式组织的,因此对表t2中的每一行,都要做100次判断,总共需要在内存中做的判断次数是:100*1000=10万次.

前面说过,如果使用Simple Nested-Loop Join算法进行查询,扫描行数也是10万行.因此,从时间复杂度上来说,这两个算法是一样的.但是Block Nested-Loop Join算法的这10万次判断是内存操作,速度上会快很多,性能也更好.

接下来,来看一下这种情况下,应该选择哪个表做驱动表.

假设小表的行数N,大表的行数是M,那么在这个算法里:
1.  两个表都做一次全表扫描,所以总的扫描行数是M+N;
2.  内存中的判断次数是:M*N.

可以看到,调换这两个算式中的M和N没差别,因此这时候选择大表还是小表做驱动表,执行耗时是一样的.


然后,这个例子里表t1才100行,要是表t1是一个大表,join_buffer放不下怎么办呢?join_buffer的大小是由参数join_buffer_size设定的,默认值是256k.如果放不下表t1的所有数据的话,策略很简单,就是分段放.这里把join_buffer_size改成1200,再执行:
```
select * from t1 straight_join t2 on (t1.a=t2.b);
```

执行过程就变成了:
1.  扫描表t1,顺序读取数据行放入join_buffer中,放完第88行join_buffer满了,继续第2步;
2.  扫描表t2,把t2中的每一行取出来,跟join_buffer中的数据对比,满足join条件的,作为结果集的一部分返回.
3.  清空join_buffer;
4.  继续扫描表t1,顺序读取最后的12行数据放入join_buffer中,继续执行第2步.

执行流程图就变成这样:
![KJbdnH.jpg](https://s2.ax1x.com/2019/10/23/KJbdnH.jpg)

图中的步骤4和5,表示清空join_buffer再复用.

可以看到,这时候由于表t1被分成了两次放入join_buffer中,导致表t2会被扫描两次.虽然分成两次放入join_buffer,但是判断等值条件的次数还是不变的,依然是(88+12)*1000=10万次.

再来看下,在这种情况下驱动表的选择问题.

假设,驱动表的数据行是N,需要分K段才能完成算法流程,被驱动表的数据行是M.注意,这里的K不是常数,N越大K就会越大,因此把K表示为y*N,显然y的取值范围是(0,1).所以,在这个算法的执行过程中:
1.  扫描的行数是 N + y * N * M;
2.  内存判断 N * M 次.

显然,内存判断次数是不收选择哪个表作为驱动表影响的.而考虑到扫描行数,在M和N大小确定的情况下,N小一些,整个算是的结果会更小.所以结论是,应该让小表当驱动表.

在 N + y * N * M 这个式子里,y才是影响扫描行数的关键因素,这个值越小越好.刚刚我们说了N越大,分段数K越大.那么,N固定的时候,join_buffer_size是K的影响因素.join_buffer_size越大,一次可以放入的行越多,分成的段数也就越少,对被驱动表的全表扫描次数就越少.这就是为什么一些建议说,如果你的join语句很慢,就把join_buffer_size改大一些.



##  总结

第一个问题:  能不能使用join语句?
1.  如果可以使用Index Nested-Loop Join算法,也就是说可以用上被驱动表上的索引,其实是没问题的;
2.  如果使用Block Nested-Loop Join算法,扫描行数就会过多.尤其是在大表上的join操作,这样可能要扫描被驱动表很多次,会占用大量的系统资源.所以这种join尽量不要用.

所以在判断要不要使用join语句时,就是看explain结果里面,Extra字段里面有没有出现"Block Nested Loop"字样.


第二个问题:  如果要使用join,应该选择大表做驱动表还是小表做驱动表?
1.  如果是Index Nested-Loop Join算法,应该选择小表做驱动
2.  如果是Block Nested-Loop Join算法:
  * 在join_buffer_size足够大的时候,是一样的;
  * 在join_buffer_size不够大的时候,应该选择小表做驱动表.