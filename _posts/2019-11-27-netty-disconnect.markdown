---
layout:     post
title:      "netty断开连接"
date:       2019-11-27 23:40:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---

#	主线

*	多路复用器(selector)接收到OP_READ事件
*	处理OP_READ事件:
	*	接收数据
	*	判断接收的数据大小是否小于0,如果是,说明是关闭,开始执行关闭:
		*	关闭channel(包括cancel多路复用器的key)
		*	清理消息: 不接收新消息,fail掉所有queue中消息
		*	触发fireChannelInactive和fireChannelUnregistered.


#	源码

断开连接的处理,是在OP_READ事件中处理完成的,处理OP_READ事件的unsafe.read()方法内部,时有如下代码段:

```
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
}
```

这里源码第4行,执行doReadBytes(byteBuf)方法时,会返回-1,因此会进入下面的if分支,然后调用byteBuf.release()做数据清理工作,并将close标志设为true.<br>
然后会跳出do...while循环,进入到后面if(close)的分支中,执行closeOnRead(pipeline)方法.进入该方法,一路跟进下去:

```
private void close(final ChannelPromise promise, final Throwable cause,
                   final ClosedChannelException closeCause, final boolean notify) {
    if (!promise.setUncancellable()) {
        return;
    }
    if (closeInitiated) {
        if (closeFuture.isDone()) {
            // Closed already.
            safeSetSuccess(promise);
        } else if (!(promise instanceof VoidChannelPromise)) { // Only needed if no VoidChannelPromise.
            // This means close() was called before so we just register a listener and return
            closeFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    promise.setSuccess();
                }
            });
        }
        return;
    }
    closeInitiated = true;
    final boolean wasActive = isActive();
    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
    Executor closeExecutor = prepareToClose();
    if (closeExecutor != null) {
        closeExecutor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    // Execute the close.
                    doClose0(promise);
                } finally {
                    // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                    invokeLater(new Runnable() {
                        @Override
                        public void run() {
                            if (outboundBuffer != null) {
                                // Fail all the queued messages
                                outboundBuffer.failFlushed(cause, notify);
                                outboundBuffer.close(closeCause);
                            }
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                }
            }
        });
    } else {
        try {
            // Close the channel and fail the queued messages in all cases.
            doClose0(promise);
        } finally {
            if (outboundBuffer != null) {
                // Fail all the queued messages.
                outboundBuffer.failFlushed(cause, notify);
                outboundBuffer.close(closeCause);
            }
        }
        if (inFlush0) {
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    fireChannelInactiveAndDeregister(wasActive);
                }
            });
        } else {
            fireChannelInactiveAndDeregister(wasActive);
        }
    }
}
```

*	this.outboundBuffer = null;	(禁止添加任何消息并刷新到outboundBuffer)