---
layout:     post
title:      "Lucene多条件查询文档号合并"
date:       2019-12-13 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - lucene

---

> 转自:Chris的博客<br>
  https://www.amazingkoala.com.cn/


#	多个MUST的QUERY的文档号合并

##	样例

这种Query组合的文档号合并的代码是在ConjunctionDISI类中实现.本文通过一个例子来介绍文档号合并逻辑.

添加10篇文档到索引中,如下图:

图1:
![Qg6bFK.png](https://s2.ax1x.com/2019/12/13/Qg6bFK.png)

使用WhiteSpaceAnalyzer进行分词.

查询条件如下图,MUST描述了满足查询要求的文档必须包含"a"、"b"、"c"、"e"四个关键字.

图2:
![Qgc0fO.png](https://s2.ax1x.com/2019/12/13/Qgc0fO.png)


##	文档号合并

将包含各个关键字的文档分别放入到各自的数组中,数组元素是文档号.

####	包含"a"的文档号
图3:
![Qgcx9U.png](https://s2.ax1x.com/2019/12/13/Qgcx9U.png)


####	包含"b"的文档号
图4:
![QggMDA.png](https://s2.ax1x.com/2019/12/13/QggMDA.png)


####	包含"c"的文档号
图5:
![Qggc2F.png](https://s2.ax1x.com/2019/12/13/Qggc2F.png)


####	包含"e"的文档号
图6:
![Qg2krj.png](https://s2.ax1x.com/2019/12/13/Qg2krj.png)


`由于满足查询要求的文档中必须都包含"a","b","c","e"四个关键字,所以满足查询要求的文档个数最多是上面几个数组中长度最小的数组的大小.`

所以合并的逻辑即遍历数组大小最小的那个,在上面的例子中,即包含"b"的文档号的数组.每次遍历一个数组元素后,再去其他数组中匹配是否包含这个文档号.遍历其他数组的顺序同样的按照数组元素大小从小到大的顺序,即包含"e"的文档号-->包含"a"的文档号-->包含"c"的文档号.


##	合并过程

从包含"b"的文档号的数组中取出第一个文档号doc1的值1,然后从包含"e"的文档号的数组中取出第一个不小于doc1(1)的文档号doc2的值,也就是5.

比较结果就是doc1(1) != doc2(5),那么没有必要继续跟其他数组作比较了.因为文档号1中不包含关键字"e".

图7:
![Qghj41.png](https://s2.ax1x.com/2019/12/13/Qghj41.png)

接着继续从包含"b"的文档号的数组中取出不小于doc2(5)的值(**<font color="red">这样取是因为,在图7的比较过程中,我们已经确定文档号1~文档号5都不同时包含关键字"b"跟"e",所以下一个比较的文档号并不是直接从包含"b"的文档号的数组中取下一个值,即2,而是根据包含"e"的文档号的数组中的doc2(5)的值,从包含"b"的文档号的数组中取出不小于5的值,即9</font>**),也就是9,doc1更新为9,然后在包含"e"的文档号的数组中取出不小于doc1(9),也就是doc2的值被更新为9:

图8:
![Qg5ZdJ.png](https://s2.ax1x.com/2019/12/13/Qg5ZdJ.png)

比较结果就是doc1(9)=doc2(9),那么我们就需要继续跟剩余的其他数组元素进行比较了,从包含"a"的文档号数组中取出不小于doc1(9)的文档号doc3的值,也就是9:

图9:
![QgIz3q.png](https://s2.ax1x.com/2019/12/13/QgIz3q.png)

这时候由于doc1(9)=doc3(9),所以需要继续跟包含"c"的文档号的数组中的元素进行比较,从包含"c"的文档号的数组中取出不小于doc1(9)的文档号doc4的值,也就是9:

图10:
![Qgomgx.png](https://s2.ax1x.com/2019/12/13/Qgomgx.png)

至此所有的数组都遍历结束,并且文档号9在所有数组中都有出现,即文档号9满足查询要求.




#	多个SHOULD文档号的合并

##	样例

添加10篇文档到索引中.如下图:  
图1:
![QHDcGj.png](https://s2.ax1x.com/2019/12/18/QHDcGj.png)

使用WhiteSpaceAnalyzer进行分词.查询条件如下图.MinimumNumberShouldMatch的值为2,表示满足查询条件的文档中必须至少包含"a","b","c","e"中的任意两个.  
图2:
![QHDxeK.png](https://s2.ax1x.com/2019/12/18/QHDxeK.png)

##	文档号合并

首先给出一个Bucket[]数组,Bucket[]数组下标为文档号,数组元素为文档出现的频率,然后分别统计包含"a","b","c","e"的文档数,将文档出现的次数写入到Bucket[]数组.

##	处理包含关键字"a"的文档

将包含"a"的文档号记录到Bucket[]数组中.  
图3:
![QHrJmV.png](https://s2.ax1x.com/2019/12/18/QHrJmV.png)

##	处理包含关键字"b"的文档

将包含"b"的文档号记录到Bucket[]数组中,文档号9第二次出现,所以计数加1.  
图4:
![QHrcTO.png](https://s2.ax1x.com/2019/12/18/QHrcTO.png)

##	处理包含关键字"c"的文档

将包含"c"的文档号记录到Bucket[]数组中,文档号3,6,8再次出现,所以对应计数都分别加1;  
图5:
![QHrLtg.png](https://s2.ax1x.com/2019/12/18/QHrLtg.png)

##	处理包含关键字"e"的文档

将包含"e"的文档号记录到Bucket[]数组中  
图6:
![QHrv1s.png](https://s2.ax1x.com/2019/12/18/QHrv1s.png)

##	统计文档号

在Bucket数组中,下标值代表了文档号,当我们处理所有关键字后,我们需要遍历文档号,然后判断每一个文档号出现的次数是否满足MinimumNumberShouldMatch,为了使得只对出现的文档号进行遍历,Lucene使用了一个matching数组记录了上面出现的文档号.matching数组记录文档号的原理跟FixedBitSet一样,都是用一个bit位来记录文档号.

在当前的例子中,我们只要用到matching[]的第一个元素,第一个元素的值为879.  
图7:
![QHyp2d.png](https://s2.ax1x.com/2019/12/18/QHyp2d.png)

根据二进制中bit位的值为1,这个bit位的位置来记录包含查询关键字的文档号,包含查询关键字的文档号只有0,1,2,3,5,6,8,9一共8篇文档,接着根据这些文档号,把他们作为bucket[]数组的下表,去找到每一个数组元素中的值,如果元素的值大于等于minShouldMatch,对应的文档就是我们最终的结果,我们的例子中.  
```
builder.setMinimumNumberShouldMatch(2);
```

所以根据最终的bucket[]  
图8:
![QH6Blj.png](https://s2.ax1x.com/2019/12/18/QH6Blj.png)

只有文档号3,文档号5,文档号6,文档号8,文档号9对应元素值大于minShouldMatch,满足查询要求