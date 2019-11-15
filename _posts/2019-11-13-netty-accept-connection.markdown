---
layout:     post
title:      "netty接收并构建连接"
date:       2019-11-12 23:15:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---


#	主线概述

###	boss thread

*	NioEventLoop中的selector轮询创建连接事件(OP_ACCEPT)
*	创建socket channel
*	初始化socket channel并从worker group中选择一个NioEventLoop


###	worker thread

*	将socket channel注册到选择的NioEventLoop的selector
*	注册读事件(OP_READ)到selector上



#	源码分析

在前面netty的启动篇,降到注册的时候,会将任务异步的丢到EventLoop中去执行,这里的逻辑是,先把任务放到taskQueue队列里,然后startThread()启动EventLoop中的线程开始执行任务.

```
public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

跟进startThread()方法进入底层,会执行doStartThread()方法

```
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }
            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }
                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    if (logger.isErrorEnabled()) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                "be called before run() implementation terminates.");
                    }
                }
                try {
                    // Run all remaining tasks and shutdown hooks.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }
                } finally {
                    try {
                        cleanup();
                    } finally {
                        FastThreadLocal.removeAll();
                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.countDown();
                        if (logger.isWarnEnabled() && !taskQueue.isEmpty()) {
                            logger.warn("An event executor terminated with " +
                                    "non-empty task queue (" + taskQueue.size() + ')');
                        }
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
```

这里上面代码第三行,会通过executor执行一个任务,先来看下这个executor是什么?他是在创建NioEventLoop的时候初始化的,其类型是ThreadPerTaskExecutor,它的executor方法,就是以Runnable作为参数,创建一个线程,然后start.

```
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;
    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }
    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

因此,doStartThread方法代码段executor.execute方法,就是开启一个线程,执行的任务就是SingleThreadEventExecutor.this.run();而这个方法,是在子类NioEventLoop中实现的.

```
protected void run() {
    for (;;) {
        try {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```