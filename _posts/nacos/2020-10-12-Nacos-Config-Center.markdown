---
layout:     post
title:      "Nacos配置中心原理----center端"
date:       2020-10-12 10:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - nacos

---

本文对Nacos配置中心center端原理进行分析.

#	一.	Servlet 3.0新特性

异步处理支持:有了该特性,Servlet线程不再需要一直阻塞,直到业务处理完毕才能再输出响应,最后才结束该Servlet线程.在接收到请求之后,Servlet线程可以将耗时的操作委派给另一个线程来完成,自己在不生成响应的情况下返回至容器.针对业务处理较耗时的情况,这将大大减少服务器资源的占用,并且提高并发处理速度.

Servlet 3.0之前,一个普通Servlet的主要工作流程大致如下:
*	首先,Servlet接收到请求之后,可能需要对请求携带的数据进行一些预处理;
*	接着,调用业务接口的某些方法,以完成业务处理;
*	最后,根据处理的结果提交响应,Servlet线程结束.

其中第二步的业务处理通常是最耗时的,这主要体现在数据库操作,以及其它的跨网络调用等,在此过程中,Servlet线程一直处于阻塞状态,直到业务方法执行完毕.在处理业务的过程中,Servlet资源一直被占用而得不到释放,对于并发较大的应用,这有可能造成性能的瓶颈.对此,在以前通常是采用私有解决方案来提前结束Servlet线程,并及时释放资源.

Servlet 3.0针对这个问题做了开创性的工作,现在通过使用Servlet 3.0的异步处理支持,之前的Servlet处理流程可以调整为如下的过程:
*	首先,Servlet接收到请求之后,可能首先需要对请求携带的数据进行一些预处理;
*	接着,Servlet线程将请求转交给一个异步线程来执行业务处理,线程本身返回至容器,此时,Servlet还没有生成响应数据.
*	异步线程处理完业务以后,可以直接生成响应数据(异步线程拥有ServletRequest 和 ServletResponse 对象的引用),或者将请求继续转发给其它Servlet.如此一来,Servlet线程不再是一直处于阻塞状态以等待业务逻辑的处理,而是启动异步线程之后可以立即返回.


**<font color="red">nacos中center端处理长轮询请求的逻辑就采用了Servlet 3.0中异步处理支持的这一新特性</font>**

#	二.	客户端长轮询配置变更监听接口

```
@PostMapping("/listener")
@Secured(action = ActionTypes.READ, parser = ConfigResourceParser.class)
public void listener(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
    request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", true);
    String probeModify = request.getParameter("Listening-Configs");
    if (StringUtils.isBlank(probeModify)) {
        throw new IllegalArgumentException("invalid probeModify");
    }
    
    probeModify = URLDecoder.decode(probeModify, Constants.ENCODE);
    
    Map<String, String> clientMd5Map;
    try {
        clientMd5Map = MD5Util.getClientMd5Map(probeModify);
    } catch (Throwable e) {
        throw new IllegalArgumentException("invalid probeModify");
    }
    
    // do long-polling
    inner.doPollingConfig(request, response, clientMd5Map, probeModify.length());
}
```

该接口逻辑比较简单,总共分为两步:
*	进行参数解析
*	执行长轮询逻辑 inner.doPollingConfig


###	二.1	 inner.doPollingConfig

```
/**
 * 轮询接口.
 */
public String doPollingConfig(HttpServletRequest request, HttpServletResponse response,
        Map<String, String> clientMd5Map, int probeRequestSize) throws IOException {
    
    // Long polling.
    if (LongPollingService.isSupportLongPolling(request)) {
        longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize);
        return HttpServletResponse.SC_OK + "";
    }
    
    // Compatible with short polling logic.
    List<String> changedGroups = MD5Util.compareMd5(request, response, clientMd5Map);
    
    // Compatible with short polling result.
    String oldResult = MD5Util.compareMd5OldResult(changedGroups);
    String newResult = MD5Util.compareMd5ResultString(changedGroups);
    
    String version = request.getHeader(Constants.CLIENT_VERSION_HEADER);
    if (version == null) {
        version = "2.0.0";
    }
    int versionNum = Protocol.getVersionNumber(version);
    
    // Befor 2.0.4 version, return value is put into header.
    if (versionNum < START_LONG_POLLING_VERSION_NUM) {
        response.addHeader(Constants.PROBE_MODIFY_RESPONSE, oldResult);
        response.addHeader(Constants.PROBE_MODIFY_RESPONSE_NEW, newResult);
    } else {
        request.setAttribute("content", newResult);
    }
    
    Loggers.AUTH.info("new content:" + newResult);
    
    // Disable cache.
    response.setHeader("Pragma", "no-cache");
    response.setDateHeader("Expires", 0);
    response.setHeader("Cache-Control", "no-cache,no-store");
    response.setStatus(HttpServletResponse.SC_OK);
    return HttpServletResponse.SC_OK + "";
}
```

这里首先会判断是否支持长轮询,支持的情况下,会进入该分支流程,进一步调用longPollingService.addLongPollingClient方法

```
public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map,
        int probeRequestSize) {
    
    String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER);
    String noHangUpFlag = req.getHeader(LongPollingService.LONG_POLLING_NO_HANG_UP_HEADER);
    String appName = req.getHeader(RequestUtil.CLIENT_APPNAME_HEADER);
    String tag = req.getHeader("Vipserver-Tag");
    int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);
    
    // Add delay time for LoadBalance, and one response is returned 500 ms in advance to avoid client timeout.
    long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
    if (isFixedPolling()) {
        timeout = Math.max(10000, getFixedPollingInterval());
        // Do nothing but set fix polling timeout.
    } else {
        long start = System.currentTimeMillis();
        List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
        if (changedGroups.size() > 0) {
            generateResponse(req, rsp, changedGroups);
            LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "instant",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
            return;
        } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
            LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "nohangup",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
            return;
        }
    }
    String ip = RequestUtil.getRemoteIp(req);
    
    // Must be called by http thread, or send response.
    final AsyncContext asyncContext = req.startAsync();
    
    // AsyncContext.setTimeout() is incorrect, Control by oneself
    asyncContext.setTimeout(0L);
    
    ConfigExecutor.executeLongPolling(
            new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
}
```
*	获得客户端传递过来的超时时间,并且进行本地计算,提前500ms返回响应,这就能解释为什么客户端响应超时时间是29.5+了
*	根据客户端请求过来的md5和服务器端对应的group下对应内容的md5进行比较,如果不一致,则通过 generateResponse 将变更列表返回
*	如果配置文件没有发生变化,则通过scheduler.execute启动了一个定时任务,将客户端的长轮询请求封装成一个叫ClientLongPolling的任务,交给scheduler去执行

<br>
那么接下去一定会进入ClientLongPolling的run方法,下面来具体看下

```
class ClientLongPolling implements Runnable {
    
    @Override
    public void run() {
        asyncTimeoutFuture = ConfigExecutor.scheduleLongPolling(new Runnable() {
            @Override
            public void run() {
                try {
                    getRetainIps().put(ClientLongPolling.this.ip, System.currentTimeMillis());
                    
                    // Delete subsciber's relations.
                    allSubs.remove(ClientLongPolling.this);
                    
                    if (isFixedPolling()) {
                        LogUtil.CLIENT_LOG
                                .info("{}|{}|{}|{}|{}|{}", (System.currentTimeMillis() - createTime), "fix",
                                        RequestUtil.getRemoteIp((HttpServletRequest) asyncContext.getRequest()),
                                        "polling", clientMd5Map.size(), probeRequestSize);
                        List<String> changedGroups = MD5Util
                                .compareMd5((HttpServletRequest) asyncContext.getRequest(),
                                        (HttpServletResponse) asyncContext.getResponse(), clientMd5Map);
                        if (changedGroups.size() > 0) {
                            sendResponse(changedGroups);
                        } else {
                            sendResponse(null);
                        }
                    } else {
                        LogUtil.CLIENT_LOG
                                .info("{}|{}|{}|{}|{}|{}", (System.currentTimeMillis() - createTime), "timeout",
                                        RequestUtil.getRemoteIp((HttpServletRequest) asyncContext.getRequest()),
                                        "polling", clientMd5Map.size(), probeRequestSize);
                        sendResponse(null);
                    }
                } catch (Throwable t) {
                    LogUtil.DEFAULT_LOG.error("long polling error:" + t.getMessage(), t.getCause());
                }
                
            }
            
        }, timeoutTime, TimeUnit.MILLISECONDS);
        
        allSubs.add(this);
    }
    
    // ......省略
}
```

这个任务要延时29.5s才会执行,因为前面已经执行过MD5比较了,不需要再执行一遍.如果在29.5s+之内,数据发生变化,需要提前通知.这里就需要有一种通知机制,此处是通过一个allSubs队列实现的.上面代码段可以看到,ClientLongPolling任务的run方法,在执行时首先将自己添加到allSubs中,后面延时29.5s执行的任务,在运行时,再先把自己从allSubs中移除.

```
/**
 * ClientLongPolling subscibers.
 */
final Queue<ClientLongPolling> allSubs;
```

而订阅机制的实现,是在调用LongPollingService构造函数进行实例创建的时候,注册发布订阅信息
```
public LongPollingService() {
    allSubs = new ConcurrentLinkedQueue<ClientLongPolling>();
    
    ConfigExecutor.scheduleLongPolling(new StatTask(), 0L, 10L, TimeUnit.SECONDS);
    
    // Register LocalDataChangeEvent to NotifyCenter.
    NotifyCenter.registerToPublisher(LocalDataChangeEvent.class, NotifyCenter.ringBufferSize);
    
    // Register A Subscriber to subscribe LocalDataChangeEvent.
    NotifyCenter.registerSubscriber(new Subscriber() {
        
        @Override
        public void onEvent(Event event) {
            if (isFixedPolling()) {
                // Ignore.
            } else {
                if (event instanceof LocalDataChangeEvent) {
                    LocalDataChangeEvent evt = (LocalDataChangeEvent) event;
                    ConfigExecutor.executeLongPolling(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
                }
            }
        }
        
        @Override
        public Class<? extends Event> subscribeType() {
            return LocalDataChangeEvent.class;
        }
    });
    
}
```

在配置变更后,会产生相应的事件LocalDataChangeEvent,走到上面代码的else分支.可以看到,这里会执行一个DataChangeTask任务,下面再来看下该任务run方法逻辑
```
class DataChangeTask implements Runnable {
    
    @Override
    public void run() {
        try {
            ConfigCacheService.getContentBetaMd5(groupKey);
            for (Iterator<ClientLongPolling> iter = allSubs.iterator(); iter.hasNext(); ) {
                ClientLongPolling clientSub = iter.next();
                if (clientSub.clientMd5Map.containsKey(groupKey)) {
                    // If published tag is not in the beta list, then it skipped.
                    if (isBeta && !CollectionUtils.contains(betaIps, clientSub.ip)) {
                        continue;
                    }
                    
                    // If published tag is not in the tag list, then it skipped.
                    if (StringUtils.isNotBlank(tag) && !tag.equals(clientSub.tag)) {
                        continue;
                    }
                    
                    getRetainIps().put(clientSub.ip, System.currentTimeMillis());
                    iter.remove(); // Delete subscribers' relationships.
                    LogUtil.CLIENT_LOG
                            .info("{}|{}|{}|{}|{}|{}|{}", (System.currentTimeMillis() - changeTime), "in-advance",
                                    RequestUtil
                                            .getRemoteIp((HttpServletRequest) clientSub.asyncContext.getRequest()),
                                    "polling", clientSub.clientMd5Map.size(), clientSub.probeRequestSize, groupKey);
                    clientSub.sendResponse(Arrays.asList(groupKey));
                }
            }
        } catch (Throwable t) {
            LogUtil.DEFAULT_LOG.error("data change error: {}", ExceptionUtil.getStackTrace(t));
        }
    }
    // ....省略
}
```

此处通过一个循环迭代器,从allSubs里面获得ClientLongPolling最后通过clientSub.sendResponse把数据返回到客户端.以此来完成数据变化实时触发更新.

#   三.  更新配置接口

```
@PostMapping
@Secured(action = ActionTypes.WRITE, parser = ConfigResourceParser.class)
public Boolean publishConfig(HttpServletRequest request, HttpServletResponse response,
        @RequestParam(value = "dataId") String dataId, @RequestParam(value = "group") String group,
        @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
        @RequestParam(value = "content") String content, @RequestParam(value = "tag", required = false) String tag,
        @RequestParam(value = "appName", required = false) String appName,
        @RequestParam(value = "src_user", required = false) String srcUser,
        @RequestParam(value = "config_tags", required = false) String configTags,
        @RequestParam(value = "desc", required = false) String desc,
        @RequestParam(value = "use", required = false) String use,
        @RequestParam(value = "effect", required = false) String effect,
        @RequestParam(value = "type", required = false) String type,
        @RequestParam(value = "schema", required = false) String schema) throws NacosException {
    
    final String srcIp = RequestUtil.getRemoteIp(request);
    final String requestIpApp = RequestUtil.getAppName(request);
    
    //  ....省略

    if (AggrWhitelist.isAggrDataId(dataId)) {
        LOGGER.warn("[aggr-conflict] {} attemp to publish single data, {}, {}", RequestUtil.getRemoteIp(request),
                dataId, group);
        throw new NacosException(NacosException.NO_RIGHT, "dataId:" + dataId + " is aggr");
    }
    
    final Timestamp time = TimeUtils.getCurrentTime();
    String betaIps = request.getHeader("betaIps");
    ConfigInfo configInfo = new ConfigInfo(dataId, group, tenant, appName, content);
    configInfo.setType(type);
    if (StringUtils.isBlank(betaIps)) {
        if (StringUtils.isBlank(tag)) {
            persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, true);
            ConfigChangePublisher
                    .notifyConfigChange(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
        } else {
            persistService.insertOrUpdateTag(configInfo, tag, srcIp, srcUser, time, true);
            ConfigChangePublisher.notifyConfigChange(
                    new ConfigDataChangeEvent(false, dataId, group, tenant, tag, time.getTime()));
        }
    } else {
        // beta publish
        persistService.insertOrUpdateBeta(configInfo, betaIps, srcIp, srcUser, time, true);
        ConfigChangePublisher
                .notifyConfigChange(new ConfigDataChangeEvent(true, dataId, group, tenant, time.getTime()));
    }
    ConfigTraceService
            .logPersistenceEvent(dataId, group, tenant, requestIpApp, time.getTime(), InetUtils.getSelfIp(),
                    ConfigTraceService.PERSISTENCE_EVENT_PUB, content);
    return true;
}
```

这里主要两个步骤:
*   persistService.insertOrUpdate----进行数据持久化存储
*   ConfigChangePublisher.notifyConfigChange----发布ConfigDataChangeEvent

#   四.  ConfigDataChangeEvent事件的处理

AsyncNotifyService的构造方法
```
public AsyncNotifyService(ServerMemberManager memberManager) {
        this.memberManager = memberManager;
        httpclient.start();
        
        // Register ConfigDataChangeEvent to NotifyCenter.
        NotifyCenter.registerToPublisher(ConfigDataChangeEvent.class, NotifyCenter.ringBufferSize);
        
        // Register A Subscriber to subscribe ConfigDataChangeEvent.
        NotifyCenter.registerSubscriber(new Subscriber() {
            
            @Override
            public void onEvent(Event event) {
                // Generate ConfigDataChangeEvent concurrently
                if (event instanceof ConfigDataChangeEvent) {
                    ConfigDataChangeEvent evt = (ConfigDataChangeEvent) event;
                    long dumpTs = evt.lastModifiedTs;
                    String dataId = evt.dataId;
                    String group = evt.group;
                    String tenant = evt.tenant;
                    String tag = evt.tag;
                    Collection<Member> ipList = memberManager.allMembers();
                    
                    // In fact, any type of queue here can be
                    Queue<NotifySingleTask> queue = new LinkedList<NotifySingleTask>();
                    for (Member member : ipList) {
                        queue.add(new NotifySingleTask(dataId, group, tenant, tag, dumpTs, member.getAddress(),
                                evt.isBeta));
                    }
                    ConfigExecutor.executeAsyncNotify(new AsyncTask(httpclient, queue));
                }
            }
            
            @Override
            public Class<? extends Event> subscribeType() {
                return ConfigDataChangeEvent.class;
            }
        });
    }
```
可以看到,AsyncNotifyService的构造方法中,订阅了对ConfigDataChangeEvent的监听,当该事件发生后,会执行onEvent方法.在该方法内部,会将cluster集群中的每个成员信息封装成一个NotifySingleTask,添加到队列中,然后构造一个异步任务以通知各个成员节点配置变更(通过http接口).

下面再来看下接受上述配置变更通知的接口逻辑 CommunicationController.notifyConfigInfo

```
public Boolean notifyConfigInfo(HttpServletRequest request, @RequestParam("dataId") String dataId,
        @RequestParam("group") String group,
        @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
        @RequestParam(value = "tag", required = false) String tag) {
    dataId = dataId.trim();
    group = group.trim();
    String lastModified = request.getHeader(NotifyService.NOTIFY_HEADER_LAST_MODIFIED);
    long lastModifiedTs = StringUtils.isEmpty(lastModified) ? -1 : Long.parseLong(lastModified);
    String handleIp = request.getHeader(NotifyService.NOTIFY_HEADER_OP_HANDLE_IP);
    String isBetaStr = request.getHeader("isBeta");
    if (StringUtils.isNotBlank(isBetaStr) && trueStr.equals(isBetaStr)) {
        dumpService.dump(dataId, group, tenant, lastModifiedTs, handleIp, true);
    } else {
        dumpService.dump(dataId, group, tenant, tag, lastModifiedTs, handleIp);
    }
    return true;
}
```
这里会通过dumpService.dump方法,将配置更新任务添加到TaskManager的任务列表中.

下面再来看下TaskManager的构造方法
```
public TaskManager(String name) {
    this.name = name;
    if (null != name && name.length() > 0) {
        this.processingThread = new Thread(new ProcessRunnable(), name);
    } else {
        this.processingThread = new Thread(new ProcessRunnable());
    }
    this.processingThread.setDaemon(true);
    this.closed.set(false);
    this.processingThread.start();
}
```

这里会启动一个线程,执行ProcessRunnable,来看下ProcessRunnable的run方法
```
class ProcessRunnable implements Runnable {
    
    @Override
    public void run() {
        while (!TaskManager.this.closed.get()) {
            try {
                Thread.sleep(100);
                TaskManager.this.process();
            } catch (Throwable e) {
                LogUtil.DUMP_LOG.error("execute dump process has error : {}", e);
            }
        }
    }
}
```
这里看到,该任务循环执行TaskManager.this.process()方法,在该方法内调用DumpProcessor的process方法,完成如下几件事:
*   查询持久化存储(通常集群模式下为mysql),获取最新配置信息
*   将最新配置信息更新到缓存中
*   发布LocalDataChangeEvent事件

这样就触发了配置变更对客户端的通知.



#	五.	客户端配置值查询接口

center缓存


#   总结

![0h2Wgs.png](https://s1.ax1x.com/2020/10/13/0h2Wgs.png)