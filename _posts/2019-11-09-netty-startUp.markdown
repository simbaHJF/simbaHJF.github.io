---
layout:     post
title:      "netty的启动"
date:       2019-11-09 13:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - netty

---



#	主线概述

###	main thread
*	创建selector
*	创建server socket channel
*	初始化server socket channel
*	给server socket channel从boss group中选择一个NioEventLoop

###	boss thread
*	将server socket channel注册到选择的NioEventLoop的selector
*	绑定地址启动
*	注册接受连接事件(OP_ACCEPT)到selector上


#	源码分析

以netty源码中example里EchoServer为例.

```
public static void main(String[] args) throws Exception {
    // Configure SSL.
    final SslContext sslCtx;
    if (SSL) {
        SelfSignedCertificate ssc = new SelfSignedCertificate();
        sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
    } else {
        sslCtx = null;
    }

    // Configure the server.
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    final EchoServerHandler serverHandler = new EchoServerHandler();
    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .option(ChannelOption.SO_BACKLOG, 100)
         .handler(new LoggingHandler(LogLevel.INFO))
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 ChannelPipeline p = ch.pipeline();
                 if (sslCtx != null) {
                     p.addLast(sslCtx.newHandler(ch.alloc()));
                 }
                 //p.addLast(new LoggingHandler(LogLevel.INFO));
                 p.addLast(serverHandler);
             }
         });

        // Start the server.
        ChannelFuture f = b.bind(PORT).sync();

        // Wait until the server socket is closed.
        f.channel().closeFuture().sync();
    } finally {
        // Shut down all event loops to terminate all threads.
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```

##	Configure the server.

new NioEventLoopGroup(1)完成的事情

这里先看一下NioEventLoopGroup和NioEventLoop的类图结构:
[![Mmmi01.md.png](https://s2.ax1x.com/2019/11/09/Mmmi01.md.png)](https://imgchr.com/i/Mmmi01)

[![MmmAk6.md.png](https://s2.ax1x.com/2019/11/09/MmmAk6.md.png)](https://imgchr.com/i/MmmAk6)


跟进方法,最后调用父类方法MultithreadEventExecutorGroup的构造方法,完成对象创建:

```
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

这里首先构造一个长度为nThreads(在创建NioEventLoopGroup时传递的参数)的数组new EventExecutor[nThreads],并通过内部的属性children来持有这个数组.也就是说NioEventLoopGroup持有的就是这一组EventExecutor.

那么这个EventExecutor具体是什么类型呢?在上面源码中,在for循环里完成了对数组元素EventExecutor的创建,具体方法由children[i] = newChild(executor, args)来实现.跟进其内部,可以看到他创建的是NioEventLoop实例.

```
@Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }
```

这样,就把NioEventLoopGroup和NioEventLoop关联起来了.关联方法就是NioEventLoopGroup内部持有一个NioEventLoop数组.

再来看一下NioEventLoop的创建过程做了哪些事情:

```
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory queueFactory) {
    super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
            rejectedExecutionHandler);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    final SelectorTuple selectorTuple = openSelector();
    selector = selectorTuple.selector;
    unwrappedSelector = selectorTuple.unwrappedSelector;
    selectStrategy = strategy;
}
```

先跟进其父类构造方法:
```
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    this.executor = ThreadExecutorMap.apply(executor, this);
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

可以看到,这里做了相应的属性设置,其中几个关键点:
*	设置parent属性为该NioEventLoop所属的NioEventLoopGroup
*	设置Executor executor执行器属性为自身,因为有上面类图可以看到,NioEventLoop继承了SingleThreadEventExecutor,这个类封装了执行任务的功能,可以理解为一个单线程任务执行器,其中内部持有一个任务队列.
*	设置Queue<Runnable> taskQueue 任务队列属性,队列最大长度默认设置为16.
*	设置RejectedExecutionHandler rejectedExecutionHandler拒绝处理器,当任务队列满时,对再提交到这个NioEventLoop中的任务,会执行这个handler中的rejected方法来拒绝任务提交执行.


父类构造方法完成,再来看下NioEventLoop自身构造方法中完成了哪些重要设置:
*	首先是设置SelectorProvider provider
*	然后通过openSelector()方法来获取一个SelectorTuple对象,该对象中的selector即为我们Java Nio编程时熟悉的selector对象.
*	设置SelectStrategy selectStrategy.该对象用于判断接下来是执行select还是直接处理IO事件和执行队列中的task.


初始化完children持有的EventExecutor数组(也就是NioEventLoop数组),后面就是chooser = chooserFactory.newChooser(children)这行代码逻辑了,它要做的事情就是以children为参数,创建一个EventExecutorChooser接口的实现,这个chooser是做什么的呢?从名字上也可以猜测,它是用来选择EventExecutor的,这个接口只有一个next()方法,是用来获取下一个可用EventExecutor的,也就是当有新的连接进来,通过这个chooser来判断将其关联到哪个EventExecutor上(在我们分析的代码这里,就是关联到哪个NioEventLoop上)


##	Start the server.

ChannelFuture f = b.bind(PORT).sync();

先来看一下b.bind(PORT)方法做了哪些事,跟进方法底层,最终调用父类方法AbstractBootstrap#doBind:

```
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

先来看下final ChannelFuture regFuture = initAndRegister()这一行,在这个方法调用中,完成了channel的创建,初始化和注册三件事,来看下源码:

```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    return regFuture;
}
```

*	首先是通过channelFactory的newChannel方法创建channel.channelFactory的类型是ReflectiveChannelFactory,是由ServerBootstrap#channel方法设置的.而newInstance方法,则是典型的工厂方法实现方式:泛型+反射:

```
@Override
public T newChannel() {
    try {
        return constructor.newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
    }
}
```

通过NioServerSocketChannel的无参构造器反射创建对象的时候,同时会通过调用父类构造器,完成一些属性的创建和设置,一些关键点包括:
*	设置NioServerSocketChannel的readInterestOp为SelectionKey.OP_ACCEPT,即关心连接事件
*	创建一个DefaultChannelPipeline对象并通过pipeline属性持有.初始化DefaultChannelPipeline对象中的头尾处理器上下文AbstractChannelHandlerContext,即head和tail,并构成一个双向链表.后面再添加处理器就是通过pipelin中的处理器双向链表进行操作的.


channel创建完成后,就是对其进行初始化操作,即init(channel)方法的调用.

```
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }
    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
    ChannelPipeline p = channel.pipeline();
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

这个方法逻辑中,前一部分没什么好说的,就是一些属性的设置.后一部分是关键,获取channel的pipeline后为其增加一个ChannelInitializer,用来给channel中添加handler.处理逻辑大体如下:
*	调用config.handler(),获取的是bootstrap.handler(),将其加入channel的pipeline中.而这个handler是在main方法中,配置bootstrap的时候,通过Bootstrap#handler方法设置的.
*	通过channel关联的NioEventLoop异步执行一个任务,该任务将ServerBootstrapAcceptor添加到channel的pipeline中.

下面来分析一下ServerBootstrapAcceptor这个对象是做什么的.

首先看一下ServerBootstrapAcceptor构造方法中传入的这几个参数,第一个是前面对应的channel(也就是NioServerSocketChannel);第二个是childGroup(也就是Bootstrap初始化时的workerGroup);第三个是childHander(Bootstrap初始化时候的childerHandler,在前面源码中,也是以一个ChannelInitializer形式体现的.);后面两个参数是一些属性设置,没什么好说.

那么ServerBootstrapAcceptor是用来做什么的呢?主要是在有新的连接请求接入到服务端时,NioServerSocketChannel对接入channel做的一些处理设置,来看一下具体逻辑:

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

大体流程如下:
*	将childHandler添加到请求连入的channel的pipeline中.这个childHandler就是Bootstrap初始化时作为ChannelInitializer传入的初始化器.
*	一些属性设置
*	将child的channel注册到childGroup(也就是workGroup)中的某个NioEventLoop上.具体方法就是MultithreadEventLoopGroup中的next().register(channel),其实现原理前面已经说过,是通过一个chooser属性持有的选择器来进行选择的.
*	为这个child的channel添加一个ChannelFutureListener,这个监听器完成的动作就是,当连入完成,结果不成功的时候,关掉这个channel.

至此,NioServerSocketChannel的初始化工作完成.接下来就是将这个channel注册到bossGroup中.也就是ChannelFuture regFuture = config().group().register(channel)这个代码完成的事情.下面来看下其内部实现:

首先,同样是通过next().register(channel)来决定将channel注册到哪个bossGroup上,这里我们初始化bossGroup的时候,传入的参数是1,因此也就只有一个boss的NioEventLoopGroup,也即我们采用的是非主从模式的Reactor多线程模式.而当我们采用主从模式的Reactor时,bossGroup初始化时,传入的参数不是1了,也就会出现多个bossGroup.

然后跟进代码,注册最终由AbstractChannel中的内部类AbstractUnsafe中的register方法来实现:

```
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

这里先看if (eventLoop.inEventLoop())这句判断,这里是判断当前线程是否是EventLoop线程,这里明显不是,而是main线程,NioEventLoop线程此时还没启动起来.因此会走else分支.在else分支中,通过一个异步任务丢到eventLoop中去执行.首先来看一下eventLoop.execute()方法.

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

大体流程是:
*	先将task假如到queue中(每个NioEventLoop中会自己内部持有一个任务queue,前面说过)
*	然后判断线程是否是EventLoop线程,否的话执行startThread()方法,启动线程来执行register任务.

下面看下startThread()方法:

```
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}
```

这里主要通过STATE_UPDATER原子的更新状态.然后调用doStartThread方法真正启动线程.看下doStartThread方法内部:

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
                logger.warn("Unexpected exception from 
            } finally {
            	......
        	}
    	}
	}
}
```



下面来看下register具体做的事,也就是register0(promise)方法:

```
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;
        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

首先来看下doRegister()方法的调用完成事情:

```
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```

这里selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this)这行来将channel注册到selector上,底层是对java中selector#register的调用.注意:这时候传递的ops参数是0,还并不是OP_ACCEPT,因为这时候还未进行bind.

在来看下doRegister()方法完成后的逻辑,会调用safeSetSuccess(promise)方法,来设置成功状态.

```
protected final void safeSetSuccess(ChannelPromise promise) {
    if (!(promise instanceof VoidChannelPromise) && !promise.trySuccess()) {
        logger.warn("Failed to mark a promise as success because it is done already: {}", promise);
    }
}
```

跟进promise.trySuccess()方法,最后会通过如下方法调用来实现:
```
private boolean setValue0(Object objResult) {
        if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
            RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
            if (checkNotifyWaiters()) {
                notifyListeners();
            }
            return true;
        }
        return false;
    }
```

这里以一个原子类AtomicReferenceFieldUpdater<DefaultPromise, Object>来更新状态,然后执行notifyListeners()方法,通知完成,唤起各监听器,执行相应处理逻辑.

至此,initAndRegister方法完成,也即NioServerSocketChannel的创建和出事话完成了,至于注册任务,已经被丢到EventLoop中去异步执行了,这时候是否完成了register,并不一定.

返回doBind方法源码,继续看下面的逻辑,根据regFuture.isDone()的结果来判断之前的register是否完成(前面说了,这个注册工作被丢到EventLoop中执行了).如果已经完成,那么直接调用doBind0(regFuture, channel, localAddress, promise)方法,执行绑定,也就是源码中的if分支流程;否则,还未register完成,那么对regFuture注册一个监听器,这个监听器的工作是,待regFuture完成后,如果成功,那么调用bind0方法进行绑定,如果失败,那么设置失败状态.

bind0方法,就是完成端口绑定的方法,来看下内部实现:

```
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

跟进channel.bind这行,最终会进入到HeadContext#bind方法的调用中(HeadContext,是pipeline中的头,前面说过,pipeline中,head,tail和其他各handler形成一个双向链表)

```
@Override
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
}
```

继续跟进:

```
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }
    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
}
```

其关键点流程如下:
*   boolean wasActive = isActive()方法获取channel是否活跃,当前并未激活,因此wasActive为false
*   调用doBind,真正进行绑定(底层java实现)
*   channel以激活,走到if (!wasActive && isActive())分支判断,进入分支流程.invokeLater方法向EventLoop提交了一个任务,也就是pipeline.fireChannelActive()方法,下面来看下这个任务要做的事情是什么:

```
public final ChannelPipeline fireChannelActive() {
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}
```

跟进底层,会调用到HeadContext#channelActive(ChannelHandlerContext ctx)方法:

```
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();
    readIfIsAutoRead();
}
```

这里readIfIsAutoRead是一个关键点,向下一直跟进,最终会调用到会调用到HeadContext#read(ChannelHandlerContext ctx)方法:

```
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```

继续跟进下去:

```
public final void beginRead() {
    assertEventLoop();
    if (!isActive()) {
        return;
    }
    try {
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```

这里的关键逻辑就是doBeginRead()方法.

```
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }
    readPending = true;
    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

到这里可以看到,interestOps是我们前面所讲的selector刚创建时注册的key,是0,并不是OP_ACCEPT;而readInterestOp是我们NioServerSocketChannel创建时候设置的key,是16,也就是OP_ACCEPT.

这两个值做位与计算,结果为0,因此会进入到if分支流程中.在分支流程中,对interestOps和readInterestOp做位或运算,得到值16,也就是OP_ACCEPT.

通过selectionKey.interestOps(interestOps | readInterestOp)这一行的执行,就已经将selector的监听事件OP_ACCEPT注册好了.



至此,Nett启动完成.另外这里再多补充一句,通过ChannelInitializer来向pipeline中添加handler时,ChannelInitializer中的initChannel方法执行完后,ChannelInitializer就会被移除了,因此ChannelInitializer是一次性的(<font color="red">注意,这里指的是ChannelInitializer是一次性的,用完移除,而并不是指通过ChannelInitializer添加来的handler,这些handler是会一直存在的</font>)