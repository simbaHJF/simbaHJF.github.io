---
layout:     post
title:      "Page Cache"
date:       2019-06-17 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - linux

---

>参考资料:<br>
深入理解Linux内核(第三版) (英文版)

##	Page Cache
Page Cache是Linux使用的主要硬盘缓存.在大多数情况下,当Linux内核从硬盘读入数据或向硬盘写入数据时,会用到Page Cache.当一个用户态进程有一个读请求时,新的页将会被加入到Page Cache中.如果也当前不再Page Cache中,一个新的entry将被添加到缓存中,这个entry由从磁盘读取的数据来填充.当空闲内存足够时,页将无时间限制的长久保持在缓存中,以便被其他进程重用,来避免访问硬盘.<br>

类似的,在写一页数据到块设备之前,内核会验证相应的页是否已经在缓存里;如果没有,一个新的entry将会被添加到缓存中,并且该entry由将要写入硬盘的数据来填充.I/O数据传输并不会立即开始:硬盘更新将会延迟一些时间,因此给进程对将要写入的数据进行进一步修改的机会(换句话说,内核实现了延迟写操作)<br>

实际上,所有的read()和write()的文件操作都依赖Page Cache.唯一的例外是:当一个进程以O_DIRECT标识来打开文件时.在这种情况下,Page Cache被绕过,I/O数据传输利用进程用户态地址空间的buffer来传输数据.一些数据库应用会利用O_DIRECT标识,这样一来他们能工使用自己的硬盘缓存算法.