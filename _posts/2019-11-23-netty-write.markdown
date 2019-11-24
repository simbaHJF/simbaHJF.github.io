---
layout:     post
title:      "netty发送数据"
date:       2019-11-23 22:40:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

>  极客时间<br>
   netty源码剖析与实战




#	写数据三种方式

![MqRhwD.png](https://s2.ax1x.com/2019/11/23/MqRhwD.png)



#	写数据的要点

*	对方仓库爆仓时,送不了的时候,会停止送,协商等通知什么时候好了,再送<br>
Netty写数据,写不进去时,会停止写,然后注册一个OP_WRITE事件,来通知什么时候可以写进去了再写.<br>
什么时候写不进去呢:
	*	对NIO编程而言,写的时候会返回一个0
	*	对BIO编程而言,会阻塞在那里

*	发送快递时,对方仓库直接收下,这个时候再发送快递时,可以尝试发送更多的快递试试,这样效果更好
netty
批量写数据时,如果想写的都写进去了,接下来的尝试写更多.

*	 发送快递时,发到某个地方的快递特别多,我们会连续发,但是快递车毕竟有限,也会考虑下其他地方.<br>
netty只有有数据要写,且能写的出去,则一直尝试,直到写不出去或者满16次(为了其他写事件也有机会得到处理,和读时候的那个16次限制,是同样的目的)

*	揽收太多,发送来不及时,爆仓,这个时候回出个告示牌:收不下了,最好过两天再来邮寄<br>
netty待写数据太多,超过一定的水位线(writeBufferWaterMark.high()),会将可写的标志位改成false,让应用端自己决定要不要发送数据了.



#	源码分析

仍然以io.netty.example.echo包下EchoServer为例.

##	write部分

发送数据write部分的处理逻辑,是从EchoServerHandler的channelRead方法开始的

```
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

从channelRead方法,一直跟进下去:

```
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ObjectUtil.checkNotNull(msg, "msg");
    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    final AbstractChannelHandlerContext next = findContextOutbound(flush ?
            (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}
```

这里,上一层调用的时候flush参数传入的是false,因此我们会进入到上面代码中第21行的代码逻辑,继续深入进去:

```
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        // If the outboundBuffer is null we know the channel was closed and so
        // need to fail the future right away. If it is not null the handling of the rest
        // will be done in flush0()
        // See https://github.com/netty/netty/issues/2362
        safeSetFailure(promise, newClosedChannelException(initialCloseCause));
        // release message now to prevent resource-leak
        ReferenceCountUtil.release(msg);
        return;
    }
    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
    outboundBuffer.addMessage(msg, size, promise);
}
```

上面代码最终会调用outboundBuffer.addMessage(msg, size, promise)方法进行消息添加,而outboundBuffer就是我们开篇要点里打比方所讲的仓库,先暂存消息:

```
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

这里的操作就是,将消息封装成一个entry,然后追加到消息链表的队尾(这里unflushedEntry指向的是消息链表的表头)



##	flush部分

发送数据的flush部分,是从EchoServerHandler的channelReadComplete方法开始的

```
public ChannelHandlerContext flush() {
    final AbstractChannelHandlerContext next = findContextOutbound(MASK_FLUSH);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeFlush();
    } else {
        Tasks tasks = next.invokeTasks;
        if (tasks == null) {
            next.invokeTasks = tasks = new Tasks(next);
        }
        safeExecute(executor, tasks.invokeFlushTask, channel().voidPromise(), null);
    }
    return this;
}
```

代码第5行,一直跟进下去,来到如下方法:

```
public final void flush() {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    outboundBuffer.addFlush();
    flush0();
}
```

这里,先获取到outboundBuffer,也就是前面我们提到的仓库,消息write时候,暂存的地方.然后调用outboundBuffer.addFlush()方法,添加到flush链表中,其实现逻辑如下:

```
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        do {
            flushed++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);
        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

这里对于unflushed链表和flushedEntry链表的操作就是,将flushedEntry链表头指向unflushedEntry链表头,表示将未flush的消息链表,添加到了flushed的链表中,然后将unflushedEntry链表置空,为了下次write的时候添加未flush的消息体.<br>
然后回到flush()方法,继续向下跟进,进入flush0()方法,一路向下调用,后面会找到doWrite(outboundBuffer)方法的调用:

```
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    SocketChannel ch = javaChannel();
    int writeSpinCount = config().getWriteSpinCount();
    do {
        if (in.isEmpty()) {
            // All written so clear OP_WRITE
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }
        // Ensure the pending writes are made of ByteBufs only.
        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        int nioBufferCnt = in.nioBufferCount();
        // Always us nioBuffers() to workaround data-corruption.
        // See https://github.com/netty/netty/issues/2761
        switch (nioBufferCnt) {
            case 0:
                // We have something else beside ByteBuffers to write so fallback to normal writes.
                writeSpinCount -= doWrite0(in);
                break;
            case 1: {
                // Only one ByteBuf so use non-gathering write
                // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                // to check if the total size of all the buffers is non-zero.
                ByteBuffer buffer = nioBuffers[0];
                int attemptedBytes = buffer.remaining();
                final int localWrittenBytes = ch.write(buffer);
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
            default: {
                // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                // to check if the total size of all the buffers is non-zero.
                // We limit the max amount to int above so cast is safe
                long attemptedBytes = in.nioBufferSize();
                final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                        maxBytesPerGatheringWrite);
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
        }
    } while (writeSpinCount > 0);
    incompleteWrite(writeSpinCount < 0);
}
```

这里进入switch块,case 1 和 default的区别在于,是否有多个数据要写,一个是单次写,一个是批量写,其内部逻辑是类似的.这里以case 1举例:<br>
*	final int localWrittenBytes = ch.write(buffer);这一行,就是调用jdk的写方法,进行写入.如果返回了0,表明暂时写不进去了,后面就会注册一个写事件OP_WRITE,下次再写,然后返回.
*	如果能写进去的话,继续向下走,调整写入大小.
*	然后调用in.removeBytes(localWrittenBytes);这个方法的作用是根据写入的结果,删掉一些已经写入的数据,同时这里面会remove一下已经写入的entry,同时调整前面flushed链表的头指针flushedEntry,以便后面的写入处理.
*	可写入次数-1,16次限制.

这里再说一下注册OP_WRITE事件后,是怎么处理的,同样是在NioEventLoop中,处理SelectionKey的时候,会有对OP_WRITE事件的处理逻辑段,其处理逻辑如下:

```
if ((readyOps & SelectionKey.OP_WRITE) != 0) {
    // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
    ch.unsafe().forceFlush();
}
```

可以看到,其实就是调用了flush.


#	总结

*	写数据写不进去时,会停止写,注册一个OP_WRITE事件,来通知什么时候可以写进去了.
*	OP_WRITE不是说有数据可写,而是说可以写进去,所以正常情况下,不能注册,否则一直触发.
*	批量写数据时,如果尝试写的都写进去了,接下来会尝试写更多
*	只要有数据要写,且能写,则一直尝试,直到16次,写16次还没写完,就直接schedule一个task来继续写,而不是用注册些时间来触发,更简洁有力.
*	待写数据太多,超过一定的水位线(writeBufferWaterMark.high()),回将可写的标志位改成false,让应用端自己做决定要不要继续写.
