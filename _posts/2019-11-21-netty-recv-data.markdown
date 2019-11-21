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

这里,上面代码第9行,就是对产生事件的SelectionKey进行处理的地方,进入方法内部,其处理读事件的代码段如下:

```
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

这里,因为已经处于workGroup中了,所以此时的channel是NioSocketChannel,看下NioSocketChannel的继承类图,可以知道,这里unsafe.read()调用的是AbstractNioByteChannel中的内部类NioByteUnsafe#read()方法,内部逻辑如下:

```
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);
    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            byteBuf = allocHandle.allocate(allocator);
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }
            allocHandle.incMessagesRead(1);
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();
        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```

这里主要流程如下:
*   获取一个ByteBufAllocator,ByteBuf的分配器
*   获取RecvByteBufAllocator.Handle,其具体实现类为AdaptiveRecvByteBufAllocator的一个内部类HandleImpl.
*   分配一个byteBuf,这里分配ByteBuf的时候,会根据每次读取的字节数多少动态调整判断下一次分配size的初始大小.
*   doReadBytes(byteBuf)读取数据到byteBuf中
*   增加读取数据次数计数
*   发布channelRead事件
*   循环判断,是否进行下一次读取
*   跳出循环后,发布ChannelReadComplete事件


这里深入看一下读数据的具体实现,也就是doReadBytes(byteBuf)的逻辑:

```
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```

跟进byteBuf.writeBytes方法:

```
public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
    ensureWritable(length);
    int writtenBytes = setBytes(writerIndex, in, length);
    if (writtenBytes > 0) {
        writerIndex += writtenBytes;
    }
    return writtenBytes;
}
```

进入setBytes方法:

```
public int setBytes(int index, ScatteringByteChannel in, int length) throws IOException {
    ensureAccessible();
    ByteBuffer tmpBuf = internalNioBuffer();
    tmpBuf.clear().position(index).limit(index + length);
    try {
        return in.read(tmpBuf);
    } catch (ClosedChannelException ignored) {
        return -1;
    }
}
```

这里就和原生java的ByteBuffer联系起来了,这里获取的是DirectByteBuffer,堆外内存.

使用对外内存的原因通常如下:
*   在进程间可以共享,减少虚拟机间的复制
*   对垃圾回收停顿的改善: 如果应用某些长期存活并大量存在的对象,经常会触发YGC或者FullGC,可以考虑报这些对象放到堆外.过大的堆会影响Java应用的性能.如果使用堆外内存的话,堆外内存是直接受操作系统管理( 而不是虚拟机).这样做的结果就是能保持一个较小的堆内内存,以减少垃圾收集对应用的影响.
*   在某些场景下可以提升程序I/O操纵的性能.少去了将数据从堆内内存拷贝到堆外内存的步骤(用户态到内核态的数据拷贝).