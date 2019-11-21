---
layout:     post
title:      "netty接受数据"
date:       2019-11-21 15:15:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---


#	主线

*	多路复用器(Selector)接收到OP_READ事件
*	分配一个初始1024字节的byte buffer来接受数据
*	从Channel接受数据到byte buffer
*	记录实际接受数据大小,调整下次分配byte buffer大小
*	触发pipeline.fireChannelRead(byteBuf)把读取到的数据传播出去
*	判断接受byte buffer是否满载而归:是,尝试继续读取直到没有数据或慢16次;否,结束本轮读取,等待下次OP_READ事件

这里解释一下为什么有16次的限制.原因很简单,就是不能总一直读一个channel数据,其他channel等久了不高兴了,避免争宠.



#	源码分析

接受数据的逻辑在workGroup中的NioEventLoop中的run方法里的for死循环里执行,具体的方法就是processSelectedKeys(),跟进该方法内部:

```
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;
        final Object a = k.attachment();
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);
            selectAgain();
            i = -1;
        }
    }
}
```