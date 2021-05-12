---
layout:     post
title:      "HashSet,LinkedHashSet,TreeSet"
date:       2021-05-12 16:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JAVA

---







## 导航
[一. 类继承关系](#jump1)
<br>
[二. 实现原理](#jump2)
<br>











<br><br>
## <span id="jump1">一. 类继承关系</span>

[![gwaTYR.png](https://z3.ax1x.com/2021/05/12/gwaTYR.png)](https://imgtu.com/i/gwaTYR)

TreeSet和HashSet间没有继承关系.<br>



<br><br>
## <span id="jump2">二. 实现原理</span>

<br>
**<font size="5">HashSet</font>** <br>

内部通过map属性持有一个HashMap,各个方法内部借助HashMap的key以及相关操作方法来实现Set功能.


<br>
**<font size="5">LinkedHashSet</font>** <br>

内部通过map属性持有一个LinkedHashMap,各个方法内部借助LinkedHashMap的key以及相关操作方法来实现Set功能.


<br>
**<font size="5">TreeSet</font>** <br>

内部通过m属性持有一个TreeMap,各个方法内部借助TreeMap的key以及相关操作方法来实现Set功能.