---
layout:     post
title:      "kafka 主题与分区"
date:       2021-06-01 11:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---




## 导航
[一. 主题、分区、副本和LOG(日志)的关系](#jump1)
<br>












<br><br>
## <span id="jump1">一. 主题、分区、副本和LOG(日志)的关系</span>

[![2nNHkF.png](https://z3.ax1x.com/2021/06/01/2nNHkF.png)](https://imgtu.com/i/2nNHkF)

主题、分区、副本和 Log (日志)的关系如上图所示，主题和分区都是提供给上层用户的抽象，而在副本层面或更加确切地说是Log层面才有实际物理上的存在。同一个分区中的多 个副本必须分布在不同的 broker 中，这样才能提供有效的数据冗余。