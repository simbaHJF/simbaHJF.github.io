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



#	三.	客户端配置值查询接口

接口地址:	/v1/cs/configs