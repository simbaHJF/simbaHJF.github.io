---
layout:     post
title:      "RocketMQ 底层基本架构思维导图"
date:       2021-04-10 15:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---









[![caWeYt.png](https://z3.ax1x.com/2021/04/10/caWeYt.png)](https://imgtu.com/i/caWeYt)



**<font size="5">RocketMQ 读写优化</font>** <br>

顺序写 + Page Cache + MMAP  <br>


MMAP有大小限制,一般约为1.5G左右,因此限制CommitLog文件大小为1G,ConsumeQueue存储30W条记录约5.72M<br>


预映射机制 + 文件预热机制:
* 内存预映射机制: Broker会针对磁盘上的各种CommitLog,ConsumeQueue文件预先分配好MappedFile,提前使用MappedByteBuffer执行map()函数完成映射,这样后续读写文件的时候,就可以直接执行了.
* 文件预热: 在提前对一些文件完成映射之后,因为映射不会直接将数据加载到内存里来,那么后续在读取CommitLog,ConsumeQueue的时候,其实有可能会频繁的从磁盘里加载数据到内存中去.为了优化这个问题,在执行完map()函数之后,会进行madvise系统调用,就是提前尽可能多的把磁盘文件加载到内存里.

