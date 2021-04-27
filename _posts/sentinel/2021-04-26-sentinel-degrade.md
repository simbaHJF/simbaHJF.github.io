---
layout:     post
title:      "Sentinel 熔断降级"
date:       2021-04-26 12:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - sentinel

---









## 导航
[Sentinel熔断降级流程](#jump1)
<br>










<br><br>
## <span id="jump1">Sentinel熔断降级流程</span>

[![gpjnXV.png](https://z3.ax1x.com/2021/04/27/gpjnXV.png)](https://imgtu.com/i/gpjnXV)


注意以下参数,同hystrix一样,当请求数小于触发熔断的最小请求数阈值时,即使异常比率超出阈值,也不会熔断
[![gpjw7D.png](https://z3.ax1x.com/2021/04/27/gpjw7D.png)](https://imgtu.com/i/gpjw7D)