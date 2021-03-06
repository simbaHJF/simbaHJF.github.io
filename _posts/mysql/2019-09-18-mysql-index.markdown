---
layout:     post
title:      "mysql索引"
date:       2019-09-18 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - mysql

---

> 极客时间--丁奇--mysql实战 学习笔记

## 导航
[一. 索引的常见类型](#jump1)
<br>
[二. B树和B+树](#jump2)
<br>
[三. MySQL InnoDB索引](#jump3)
<br>
[四. 索引使用的注意事项](#jump4)
<br>










<br><br>
## <span id="jump1">一. 索引的常见类型</span>

1.	哈希表	一种键-值(key-value)存储数据的结构,当key发生哈希碰撞时,通常采用链地址法解决.
*	哈希索引做区间查询的速度是很慢的,哈希表这种结构适用于只有等值查询的场景.
*	Hash索引无法被用来避免数据的排序操作.
*	Hash索引不能利用部分索引键查询&emsp;&emsp;对于组合索引,Hash索引在计算Hash值的时候是组合索引键合并后再一起计算Hash值,而不是单独计算Hash 值,所以通过组合索引的前面一个或几个索引键进行查询的时候,Hash索引也无法被利用.
*	Hash索引遇到大量Hash值相等的情况后性能并不一定就会比B-Tree索引高.

2.	有序数组	有序数组在等值查询和范围查询场景中的性能读非常优秀.但是,在需要更新数据的时候就很麻烦,往中间插入一个记录就必须得挪动后面所有的记录,成本太高.
所以,有序数组索引只适用于静态存储引擎.比如要保存2017年某个城市的所有人口信息,这类不会再修改的数据.

3.	二叉搜索树	为了维持O(log(N))的查询复杂度,就需要保持这棵树是平衡二叉树,这就造成了更新的复杂性.
另外,假设有一个100万节点的平衡二叉树,树高20.一次查询可能需要访问20个数据块.在机械硬盘时代,从磁盘随机读一个数据块需要10ms左右的寻址时间.也就是说,对于一个100万行的表,如果使用二叉树来存储,单独访问一个行可能需要20个10ms的时间,这个查询是很慢的.
为了让一个查询尽量少地读磁盘,就必须让查询过程访问尽量少的数据块.那么,我们就不应该使用二叉树,而是要使用"N叉"树.


以InnoDB的一个整数字段索引为例,这个N差不多是1200.这棵树高是4的时候,就可以存1200的3次方个值,大约17亿.考虑到树根的数据块总是在内存中的,一个10亿行的表上的一个整数字段索引,查找一个值最多只需要访问3次磁盘.其实,树的第二层也有很大概率在内存中,那么访问磁盘的平均次数就更少了.


索引|MyISAM引擎|InnoDB引擎|Memory引擎
-|-|-|-
B-Tree索引|支持|支持|支持
HASH索引|不支持|不支持|支持

比较常用的索引是B-Tree索引和Hash索引,只有Memory引擎支持Hash索引.Hash索引适用于Key-Value查询,通过Hash索引要比通过B-Tree索引查询更迅速;Hash索引不适用范围查询,例如<、>、<=、>=这类操作.如果使用Memory引擎并且where条件中不使用"="进行索引列,那么不会用到索引.Memory引擎只有在"="的条件下才会使用索引.



<br><br>
## <span id="jump2">二. B树和B+树</span>


<br>
**<font size="4">B树</font>** <br>

B树是一种多路自平衡搜索树,它类似普通的二叉树,但是B树允许每个节点有更多的子节点.B树示意图如下:
![ueM0df.png](https://s2.ax1x.com/2019/09/25/ueM0df.png)
B树的特点:
*	所有键值分布在整个树中
*	任何关键字出现且只出现在一个节点中
*	搜索有可能在非叶子节点结束
*	在关键字全集内做一次查找,性能逼近二分查找算法


<br>
**<font size="4">B+树</font>** <br>

B+树是B树的变体,也是一种多路平衡查找树,B+树的示意图为:
![ueQ800.png](https://s2.ax1x.com/2019/09/25/ueQ800.png)
B+树与B树的不同在于:
*	所有关键字存储在叶子节点,非叶子节点不存储真正的data.这样非叶子节点就能存储更多的索引数据,索引更大范围,降低树的高度,减少IO次数
*	为所有叶子节点增加一个链指针.这样范围查找只需要通过指针来遍历就可以.



<br><br>
## <span id="jump3">三. MySQL InnoDB索引</span>

MySQL中InnoDB存储引擎使用B+树作为索引模型.引擎中表根据主键顺序以索引的形式存放,这种存储方式的表称为索引组织表.
索引类型分为主键索引和非主键索引.主键索引的叶子节点存放的是整行数据,主键索引也被称为聚簇索引;非主键索引的叶子节点存放的是主键的值和索引键值,非主键索引也被称为二级索引.

假设现在有表:

```
mysql> create table T(
id int primary key, 
k int not null, 
m int not null,
name varchar(16),
index (k)
)engine=InnoDB;
```

表中R1\~R5的(ID,k)值分别为(100,1),(200,2),(300,3),(500,5),(600,6),两棵树的示例示意图如下:
[![njR2kT.png](https://s2.ax1x.com/2019/09/20/njR2kT.png)](https://imgchr.com/i/njR2kT)
根据上面的索引结构,我们来看一下:基于主键索引和普通索引的查询有什么区别
*	如果语句是 select * from T where ID = 500,即主键查询方式,则只需要搜索ID这棵B+树;
*	如果语句是 select * from T where k = 5,即普通索引查询方式,则需要先搜索k索引树,得到ID的值为500,再到ID索引树搜索一次.这个过程称为回表.

也就是说,基于非主键索引的查询需要多扫描一棵索引树,效率没有主键索引高.当然,也存在这样的场景,就是需要根据非主键索引字段进行查询,且不需要返回整行数据,比如上面的表中,查询语句为:select id from T where k = 10,此时非主键索引中包含了要查询的所有字段值,就不会再做回表操作,这种查询方式称为覆盖索引.


<br>
**<font size="4">覆盖索引</font>** <br>

假设现在有T表
```
mysql> create table T (
ID int primary key,
k int NOT NULL DEFAULT 0, 
s varchar(16) NOT NULL DEFAULT '',
index k(k)
engine=InnoDB;

insert into T values(100,1, 'aa'),(200,2,'bb'),(300,3,'cc'),(500,5,'ee'),(600,6,'ff'),(700,7,'gg');
```

在T表中,如果执行 select * from T where k between 3 and 5,需要执行几次树的搜索操作,会扫描多少行?

表T的索引组织结构
![uPZqIK.png](https://s2.ax1x.com/2019/09/23/uPZqIK.png)

这条SQL查询语句的执行流程:
1.	在k索引树上找到k=3的记录,取得ID=300;
2.	再到ID索引树查到ID=300对应的R3;
3.	在k索引树取下一个值k=5,取得ID=500;
4.	再回到ID索引树查到ID=500对应的R4;
5.	在k索引树取下一个值k=6,不满足条件,循环结束.

在这个过程中,回到主键索引树搜索的过程,称为回表.这个查询过程读了k索引树的3条记录(步骤1,3,5),回表了两次(步骤2和4).

如果执行的语句是select ID from T where k between 3 and 5,这时只需要查ID的值,而ID的值已经在k索引树上了,因此可以直接提供查询结果,不需要回表.也就是说在这个查询里面,索引k已经"覆盖了"查询需求,称为覆盖索引.由于覆盖索引可以减少树的搜索次数,显著提升查询性能,所以使用覆盖索引是一个常用的性能优化手段.


基于上面覆盖索引的说明,来讨论一个问题:在一个市民信息表上,是否有必要将身份证号和名字建立联合索引?假设这个市民表的定义如下:
```
CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card` (`id_card`),
  KEY `name_age` (`name`,`age`)
) ENGINE=InnoDB
```

身份证号是市民的唯一标识,如果有根据身份证号查询市民信息的需求,只要在身份证号字段上建立索引就够了.而再建立一个(身份证号,姓名)的联合索引,是否浪费空间?

如果现在有一个高频请求,要根据市民的身份证号查询他的姓名,这个联合索引就有意义了.它可以在这个高频请求上用到覆盖索引,不再需要回表查整行记录,减少语句的执行时间.

当然,索引字段的维护总是有代价的.因此,在建立冗余索引来支持覆盖索引时就需要权衡考虑.


<br>
**<font size="4">最左匹配原则</font>** <br>

B+树的索引结构,可以利用索引的"最左匹配",来定位记录.为了直观地说明这个概念,用(name,age)这个联合索引来分析.
![uPcy90.jpg](https://s2.ax1x.com/2019/09/23/uPcy90.jpg)

索引项是按照**<font color="red">索引定义</font>**里面出现的字段顺序排序的.当需要查到所有名字是"张三"的人时,可以快速定位到ID4,然后向后遍历得到所有需要的结果.

如果要查的是所有名字第一个字是"张"的人,SQL语句的条件时"where name like '张%'".这时,也能够用上这个索引,查找到第一个符合条件的记录是ID3,然后向后遍历,直到不满足条件为止.

可以看到,不只是索引的全部定义,只要满足最左前缀,就可以利用索引来加速检索.这个最左前缀可以使联合索引的最左N个字段,也可以是字符串索引的最左M个字符.

基于上面对最左前缀索引的说明,来讨论一个问题:在建立联合索引的时候,如何安排索引内的字段顺序.

这里的评估标准是,索引的复用能力.因为可以支持最左前缀,所以当已经有了(a,b)这个联合索引后,一般就不需要单独在a上建立索引了.因此,第一原则是,如果通过调整顺序,可以少维护一个索引,那么这个顺序往往就是需要优先考虑采用的.

如果既有联合查询,又有基于a,b各自的查询的情况下.比如说查询条件里面只有b的语句,是无法使用(a,b)这个联合索引的,这时候需要维护另外一个索引,也就是说需要同时维护(a,b)(b)这两个索引.这时候,要考虑的原则就是空间了.比如上面的市民表的情况,name字段是比age字段大的,那么适合的索引是创建一个(name,age)的联合索引和一个(age)的单字段索引.


<br>
**<font size="4">索引下推</font>** <br>

仍然以市民表的联合索引(name,age)为例.如果现在有一个需求:检索出表中"名字第一个字是张,而且年龄是10岁的所有男孩".那么,SQL语句是这么写的:
```
mysql> select * from tuser where name like '张 %' and age=10 and ismale=1;
```

根据前缀索引规则,这个语句在搜索索引树的时候,只能用"张",找到第一个满足条件的记录ID3.当然,这还不错,总比全表扫描要好.然后会判断其他条件是否满足.

在MySQL5.6之前,只能从ID3开始一个个回表.到主键索引上找出数据行,再对比字段值.

在MySQL5.6引入了索引下推优化,可以在索引遍历过程中,对索引中包含的字段先做判断,直接过滤掉不满足条件的记录,减少回表次数.

无索引下推执行流程:
![uiEnBt.jpg](https://s2.ax1x.com/2019/09/23/uiEnBt.jpg)

这个过程InnoDB并不会去看age的值,只是按顺序把"name第一个字是'张'"的记录一条条取出来回表.因此,需要回表4次.图中每一个虚线箭头表示回表一次.

索引下推执行流程:
![uimNzF.jpg](https://s2.ax1x.com/2019/09/23/uimNzF.jpg)

InnoDB在(name,age)索引内部就判断了age是否等于10,对于不等于10的记录,直接判断并跳过.在这个例子中,只需要对ID4,ID5这两条记录回表取数据判断,就只需要回表2次.图中每一个虚线箭头表示回表一次.<br>

<font color="red">总结一句就是,索引下推就是,过滤的过程,如果非主键索引中有过滤字段,就直接的过滤掉了,不再去回表主键索引查询再过滤.这个看起来理所当然的流程,没想到居然5.6版本才作为新特性支持,amazing....</font>



<br>
## <span id="jump4">四. 索引使用的注意事项</span>


<br>
**<font size="4">条件字段函数操作</font>** <br>

如果对索引字段做了函数计算,就用不上索引了,这是MySQL的规定.对索引字段做函数操作,可能会破坏索引值的有序性,因此优化器就决定放弃走树搜索功能.需要注意的是,优化器并不是放弃使用索引,放弃树搜索功能,优化器可以选择遍历主键索引,也可以选择遍历普通索引.


<br>
**<font size="4">隐式类型转换</font>** <br>
MySQL在int类型和字符串类型做比较时,会隐式的将字符串转化为int型.因此,这时候也无法应用索引了.