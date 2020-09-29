---
layout:     post
title:      "Nacos配置中心原理----client端"
date:       2020-09-29 00:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 微服务

---


> <font color="red"> nacos版本:	1.3.2 </font>


本文对Nacos配置中心客户端原理进行分析,以nacos官方文档,java SDK的方式为基础.


先看下官方文档给出的示例代码:

```
String serverAddr = "{serverAddr}";
String dataId = "{dataId}";
String group = "{group}";
Properties properties = new Properties();
properties.put("serverAddr", serverAddr);
ConfigService configService = NacosFactory.createConfigService(properties);
String content = configService.getConfig(dataId, group, 5000);
System.out.println(content);
configService.addListener(dataId, group, new Listener() {
	@Override
	public void receiveConfigInfo(String configInfo) {
		System.out.println("recieve1:" + configInfo);
	}
	@Override
	public Executor getExecutor() {
		return null;
	}
});

// 测试让主线程不退出，因为订阅配置是守护线程，主线程退出守护线程就会退出。 正式代码中无需下面代码
while (true) {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

这里主要分三步:
*	创建ConfigService对象
*	获取配置
*	添加监听器

下面来具体分析


###		一		&emsp;&emsp;	NacosFactory.createConfigService(properties);

来看下NacosFactory.createConfigService(properties);方法的内部实现

```
public static ConfigService createConfigService(Properties properties) throws NacosException {
    return ConfigFactory.createConfigService(properties);
}
```

继续跟进
```
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable e) {
        throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
    }
}
```

这里我们看到,以反射的方式创建了一个NacosConfigService实例,再来看下NacosConfigService的构造方法:

```
public NacosConfigService(Properties properties) throws NacosException {
    ValidatorUtils.checkInitParam(properties);
    String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
    if (StringUtils.isBlank(encodeTmp)) {
        this.encode = Constants.ENCODE;
    } else {
        this.encode = encodeTmp.trim();
    }
    initNamespace(properties);
    
    this.agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
    this.agent.start();
    this.worker = new ClientWorker(this.agent, this.configFilterChainManager, properties);
}
```
这里完成如下几件事:
*	进行编码设置
*	初始化命名空间(如果没有设置的话,这块将会是空字符串)
*	设置http代理,也就是agent;并启动agent
*	创建ClientWorker(Longpolling,长轮询)

前两步很简单,没什么好说的,这里主要说下后面两步.


####	一.1  	&emsp;&emsp;  设置http代理,启动agent

设置http代理,启动agent有如下两行代码来完成:

```
agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
agent.start();
```

关注几个类HttpAgent,MetricsHttpAgent,ServerHttpAgent.

HttpAgent是一个接口,而MetricsHttpAgent,ServerHttpAgent则是这个接口的实现类.  
HttpAgent抽象了各个http请求方法,是nacos config的client中一个http代理的顶层抽象,
```
public interface HttpAgent {
    /**
     * start to get nacos ip list.
     */
    void start() throws NacosException;
    
    /**
     * invoke http get method.
     *
     * @param path          http path
     * @param headers       http headers
     * @param paramValues   http paramValues http
     * @param encoding      http encode
     * @param readTimeoutMs http timeout
     * @return HttpResult http response
     * @throws Exception If an input or output exception occurred
     */
    
    HttpRestResult<String> httpGet(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    /**
     * invoke http post method.
     *
     * @param path          http path
     * @param headers       http headers
     * @param paramValues   http paramValues http
     * @param encoding      http encode
     * @param readTimeoutMs http timeout
     * @return HttpResult http response
     * @throws Exception If an input or output exception occurred
     */
    HttpRestResult<String> httpPost(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    /**
     * invoke http delete method.
     *
     * @param path          http path
     * @param headers       http headers
     * @param paramValues   http paramValues http
     * @param encoding      http encode
     * @param readTimeoutMs http timeout
     * @return HttpResult http response
     * @throws Exception If an input or output exception occurred
     */
    HttpRestResult<String> httpDelete(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    /**
     * get name.
     */
    String getName();
    
    /**
     * get namespace.
     */
    String getNamespace();
    
    /**
     * get tenant.
     */
    String getTenant();
    
    /**
     * get encode.
     */
    String getEncode();
}
```

再看下上层代码中agent = new MetricsHttpAgent(new ServerHttpAgent(properties));这一行以及MetricsHttpAgent的构造方法:
```
public MetricsHttpAgent(HttpAgent httpAgent) {
    this.httpAgent = httpAgent;
}
```

这里,我们看到,MetricsHttpAgent内部持有了一个ServerHttpAgent,并通过包装ServerHttpAgent的各个方法完成相关操作.

那么再看一下ServerHttpAgent的构造方法:
```
public ServerHttpAgent(Properties properties) throws NacosException {
    this.serverListMgr = new ServerListManager(properties);
    this.securityProxy = new SecurityProxy(properties, NACOS_RESTTEMPLATE);
    this.namespaceId = properties.getProperty(PropertyKeyConst.NAMESPACE);
    init(properties);
    this.securityProxy.login(this.serverListMgr.getServerUrls());
    
    // init executorService
    this.executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.config.security.updater");
            t.setDaemon(true);
            return t;
        }
    });
    
    this.executorService.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            securityProxy.login(serverListMgr.getServerUrls());
        }
    }, 0, this.securityInfoRefreshIntervalMills, TimeUnit.MILLISECONDS);
    
}
```
这里主要就是创建一个ServerListManager,内部会初始化一些和nacos server相关的属性;  
init(properties)会初始化其他一些属性.  
这里看一下ServerListManager的构造方法中做的事情:
```
public ServerListManager(Properties properties) throws NacosException {
    this.isStarted = false;
    this.serverAddrsStr = properties.getProperty(PropertyKeyConst.SERVER_ADDR);
    String namespace = properties.getProperty(PropertyKeyConst.NAMESPACE);
    initParam(properties);
    
    if (StringUtils.isNotEmpty(serverAddrsStr)) {
        this.isFixed = true;
        List<String> serverAddrs = new ArrayList<String>();
        String[] serverAddrsArr = this.serverAddrsStr.split(",");
        for (String serverAddr : serverAddrsArr) {
            if (serverAddr.startsWith(HTTPS) || serverAddr.startsWith(HTTP)) {
                serverAddrs.add(serverAddr);
            } else {
                String[] serverAddrArr = serverAddr.split(":");
                if (serverAddrArr.length == 1) {
                    serverAddrs.add(HTTP + serverAddrArr[0] + ":" + ParamUtil.getDefaultServerPort());
                } else {
                    serverAddrs.add(HTTP + serverAddr);
                }
            }
        }
        this.serverUrls = serverAddrs;
        if (StringUtils.isBlank(namespace)) {
            this.name = FIXED_NAME + "-" + getFixedNameSuffix(
                    this.serverUrls.toArray(new String[this.serverUrls.size()]));
        } else {
            this.namespace = namespace;
            this.tenant = namespace;
            this.name = FIXED_NAME + "-" + getFixedNameSuffix(
                    this.serverUrls.toArray(new String[this.serverUrls.size()])) + "-" + namespace;
        }
    } else {
        if (StringUtils.isBlank(endpoint)) {
            throw new NacosException(NacosException.CLIENT_INVALID_PARAM, "endpoint is blank");
        }
        this.isFixed = false;
        if (StringUtils.isBlank(namespace)) {
            this.name = endpoint;
            this.addressServerUrl = String
                    .format("http://%s:%d/%s/%s", this.endpoint, this.endpointPort, this.contentPath,
                            this.serverListName);
        } else {
            this.namespace = namespace;
            this.tenant = namespace;
            this.name = this.endpoint + "-" + namespace;
            this.addressServerUrl = String
                    .format("http://%s:%d/%s/%s?namespace=%s", this.endpoint, this.endpointPort, this.contentPath,
                            this.serverListName, namespace);
        }
    }
}
```
这里主要就是获取server地址,namespace,endpoint等属性并设置.  

如果设置了serverAddr属性,那么isFixed置为true,表示server地址固定;否则isFixed置为false,表示不固定.  

这里再解释一下endpoint和serverAddr的作用:  

*   获取并解析endpoint是为了让客户端可以敏锐的感知到服务端集群的变化(扩容,下线),利用本地配置文件Properties获取的话需要重启服务器不够友好 ,所以我们可以通过一个专门的服务来提供集群列表的数据;这个就是endpoint.客户端定时的去请求这个endpoint获取最新的集群列表.

*   如果配置了属性serverAddr; 那么endpoint就不会生效;客户端会优先读取serverAddr的集群列表;
*   serverAddr和endpoint必须配置一个,都不配置会抛出异常

另外这里再提一下tenant属性,tenant属性默认为空.如果客户端启动的时候设置了namespace,那么tenant也将会被设置为与namespace相同的字符串值;否则namespace和tenant都是默认为空串.




说完了MetricsHttpAgent及ServerHttpAgent的创建,下面就是agent.start()方法了,来看下源码:
```
public void start() throws NacosException {
    httpAgent.start();
}
```

由前面的分析可知,agent即是构造方法中传入的ServerHttpAgent,因此这里最终会调用ServerHttpAgent的start()方法:
```
public synchronized void start() throws NacosException {
    serverListMgr.start();
}
```

到这里我们看到,会调用前面初始化的ServerListManager的start方法:
```
public synchronized void start() throws NacosException {
    
    if (isStarted || isFixed) {
        return;
    }
    
    GetServerListTask getServersTask = new GetServerListTask(addressServerUrl);
    for (int i = 0; i < initServerlistRetryTimes && serverUrls.isEmpty(); ++i) {
        getServersTask.run();
        try {
            this.wait((i + 1) * 100L);
        } catch (Exception e) {
            LOGGER.warn("get serverlist fail,url: {}", addressServerUrl);
        }
    }
    
    if (serverUrls.isEmpty()) {
        LOGGER.error("[init-serverlist] fail to get NACOS-server serverlist! env: {}, url: {}", name,
                addressServerUrl);
        throw new NacosException(NacosException.SERVER_ERROR,
                "fail to get NACOS-server serverlist! env:" + name + ", not connnect url:" + addressServerUrl);
    }
    
    // executor schedules the timer task
    this.executorService.scheduleWithFixedDelay(getServersTask, 0L, 30L, TimeUnit.SECONDS);
    isStarted = true;
}
```

由于前面我们设置了serverAddr,在初始化ServerListManager时,会将isStarted置为false,而isFixed置为true.所以这里在if判断分支后,就直接返回了.

如果没有设置serverAddr,那么isFixed为false,也就是走endpoint.那么会进行到if分支后面的代码.也就是启动一个定时任务GetServerListTask,每30秒执行一次,它的功能是:
*   请求addressServerUrl获取列表;
*   如果集群有变化;则更新列表serverUrls
*   如果集群有变化,更新属性currentServerAddr,这个表示当前客户端需要连接的服务端集群中的某个服务;具体操作是创建一个随机遍历器iterator;然后currentServerAddr = iterator.next();


到这里,设置http代理及启动agent说完了.


####    一.2   &emsp;&emsp;  创建ClientWorker

```
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,
            final Properties properties) {
        this.agent = agent;
        this.configFilterChainManager = configFilterChainManager;
        
        // Initialize the timeout parameter
        
        init(properties);
        
        this.executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });
        
        this.executorService = Executors
                .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r);
                        t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                        t.setDaemon(true);
                        return t;
                    }
                });
        
        this.executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);
    }
```

这里首先做的是agent和configFilterChainManager的属性设置.这里由名字也不难看到,ConfigFilterChainManager是一个典型的责任链模式,用于请求的责任链式过滤
然后就是在init方法中初始化一些其他配置.  
然后就是创建了两个线程池:
*   第一个线程池是只拥有一个线程,用来执行定时任务的executor,executor每隔10ms就会执行一次checkConfigInfo() 方法,从方法名上可以知道是每10ms检查一次配置信息.
*   第二个线程池是一个普通的线程池,从ThreadFactory的名称可以看到这个线程池是做长轮询的.

#####   一.2.1       checkConfigInfo
现在我们来看下checkConfigInfo()方法:
```
public void checkConfigInfo() {
    // 分任务
    int listenerSize = cacheMap.get().size();
    // 向上取整为批数
    int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
    if (longingTaskCount > currentLongingTaskCount) {
        for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
            // 要判断任务是否在执行 这块需要好好想想。 任务列表现在是无序的。变化过程可能有问题
            executorService.execute(new LongPollingRunnable(i));
        }
        currentLongingTaskCount = longingTaskCount;
    }
}
```

这里主要就是将监听配置分割为多个任务,然后丢到前面提到过的另外一个线程池executorService中来执行.这里稍微说一句cacheMap,在对某个dataId添加监听器的时候,会将其添加到该cacheMap中,后文讲configService.addListener的时候详细说.  

#####   一.2.2       LongPollingRunnable   
这里先重点看下要执行的任务,也就是LongPollingRunnable,在分析该任务源码前,应该先查看下第三部分configService.addListener的源码分析,因为这里的长轮询任务内部逻辑,是在已经对某些dateId添加了监听器的前提下执行的,不然会搞不清前后逻辑依赖,容易懵逼.

```
class LongPollingRunnable implements Runnable {
    
    private final int taskId;
    
    public LongPollingRunnable(int taskId) {
        this.taskId = taskId;
    }
    
    @Override
    public void run() {
        
        List<CacheData> cacheDatas = new ArrayList<CacheData>();
        List<String> inInitializingCacheList = new ArrayList<String>();
        try {
            // check failover config
            for (CacheData cacheData : cacheMap.get().values()) {
                if (cacheData.getTaskId() == taskId) {
                    cacheDatas.add(cacheData);
                    try {
                        checkLocalConfig(cacheData);
                        if (cacheData.isUseLocalConfigInfo()) {
                            cacheData.checkListenerMd5();
                        }
                    } catch (Exception e) {
                        LOGGER.error("get local config info error", e);
                    }
                }
            }
            
            // check server config
            List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
            if (!CollectionUtils.isEmpty(changedGroupKeys)) {
                LOGGER.info("get changedGroupKeys:" + changedGroupKeys);
            }
            
            for (String groupKey : changedGroupKeys) {
                String[] key = GroupKey.parseKey(groupKey);
                String dataId = key[0];
                String group = key[1];
                String tenant = null;
                if (key.length == 3) {
                    tenant = key[2];
                }
                try {
                    String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                    CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                    cache.setContent(ct[0]);
                    if (null != ct[1]) {
                        cache.setType(ct[1]);
                    }
                    LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}, type={}",
                            agent.getName(), dataId, group, tenant, cache.getMd5(),
                            ContentUtils.truncateContent(ct[0]), ct[1]);
                } catch (NacosException ioe) {
                    String message = String
                            .format("[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                                    agent.getName(), dataId, group, tenant);
                    LOGGER.error(message, ioe);
                }
            }
            for (CacheData cacheData : cacheDatas) {
                if (!cacheData.isInitializing() || inInitializingCacheList
                        .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                    cacheData.checkListenerMd5();
                    cacheData.setInitializing(false);
                }
            }
            inInitializingCacheList.clear();
            
            executorService.execute(this);
            
        } catch (Throwable e) {
            
            // If the rotation training task is abnormal, the next execution time of the task will be punished
            LOGGER.error("longPolling error : ", e);
            executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
        }
    }
}
```

这里主要分三个部分:
1.  check failover config.这里检查的是failover文件,而不是snapshot文件,注意这点就行了,另外我本地验证的时候,是没有生成过failover路径对应的文件的,不确定是没开启,还是版本问题
2.  check server config.从Server获取值变化了的DataID列表,返回的是一个key列表,key由dataId,group和tenant(如果有的话)构成.
3.  遍历第2步返回的有变化的key列表,对每一个key,反向解析出dataId,group,tenant,然后调用getServerConfig方法,获取配置的content,并更新md5
4.  对变更了的CacheData,调用cacheData.checkListenerMd5(),该方法内部执行监听器回调方法,拉起回调是通过listener.receiveConfigInfo(contentTmp);这一行代码实现,由此可见,回调执行的是Listener的receiveConfigInfo方法,然后对listenerWrap的lastCallMd5进行更新.

这里第3点再多说两句:  
一个是:正确收到服务端的响应结果后,会将配置的content内容更新到本地snapshot文件中.  
另一个是:获取到content后会调用cache.setContent(content),在setContent方法内部,更新content后,还会继而更新CacheData对象的md5属性.


完成上述操作后,会执行executorService.execute(this);方法,开启下一次长轮询.



###		二.  &emsp;&emsp;  configService.getConfig(dataId, group, 5000);

跟进方法底层,如下:
```
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = null2defaultGroup(group);
    ParamUtils.checkKeyParam(dataId, group);
    ConfigResponse cr = new ConfigResponse();
    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);
    // 优先使用本地配置
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
            dataId, group, tenant, ContentUtils.truncateContent(content));
        cr.setContent(content);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    }
    try {
        content = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        cr.setContent(content);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    } catch (NacosException ioe) {
        if (NacosException.NO_RIGHT == ioe.getErrCode()) {
            throw ioe;
        }
        LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
            agent.getName(), dataId, group, tenant, ioe.toString());
    }
    LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
        dataId, group, tenant, ContentUtils.truncateContent(content));
    content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
    cr.setContent(content);
    configFilterChainManager.doFilter(null, cr);
    content = cr.getContent();
    return content;
}
```

这里主要三个关键点:
1.  这里首先会优先使用本地配置,也就是failover对应路径下的文件,如果存在就直接返回了,但是我在验证的时候,没见到有任何地方和代码逻辑会生成这个failover文件.  
2.  如果不存在failover对应文件内容,就会去请求server端来获取配置内容
3.  如果第二步过程中抛出了异常,会获取snapshot文件中的内容作为配置返回.

这部分代码逻辑比较简单,就不深入说了.


###		三.  &emsp;&emsp;  configService.addListener

跟进configService.addListener内部代码实现:

```
public void addTenantListeners(String dataId, String group, List<? extends Listener> listeners)
        throws NacosException {
    group = null2defaultGroup(group);
    String tenant = agent.getTenant();
    CacheData cache = addCacheDataIfAbsent(dataId, group, tenant);
    for (Listener listener : listeners) {
        cache.addListener(listener);
    }
}
```

这里关键点有2个:
*   addCacheDataIfAbsent方法
*   cache.addListener

下面逐个来分析

####    三.1   &emsp;&emsp;    addCacheDataIfAbsent方法

来看下其内部实现:

```
public CacheData addCacheDataIfAbsent(String dataId, String group, String tenant) throws NacosException {
    // 首先尝试从cacheMap中获取,如果不为空,直接返回
    CacheData cache = getCache(dataId, group, tenant);
    if (null != cache) {
        return cache;
    }
    // 根据dataId,group,tenant生成key
    String key = GroupKey.getKeyTenant(dataId, group, tenant);
    synchronized (cacheMap) {
        CacheData cacheFromMap = getCache(dataId, group, tenant);
        // multiple listeners on the same dataid+group and race condition,so
        // double check again
        // other listener thread beat me to set to 
        //  双重加锁方式,这里有并发的情况,其他线程为相同dataid+group的CacheData添加了其他listener
        if (null != cacheFromMap) {
            cache = cacheFromMap;
            // reset so that server not hang this check
            cache.setInitializing(true);
        } else {
            cache = new CacheData(configFilterChainManager, agent.getName(), dataId, group, tenant);
            // fix issue # 1317
            // 如果为true,直接从远程服务器进行配置获取
            if (enableRemoteSyncConfig) {
                String content = getServerConfig(dataId, group, tenant, 3000L);
                cache.setContent(content);
            }
        }
        // 复制一份当前的cacheMap为copy,然后向其中添加刚刚构造的CacheData对象,然后替换掉原来的cacheMap
        // 这里cacheMap是一个AtomicReference.
        Map<String, CacheData> copy = new HashMap<String, CacheData>(cacheMap.get());
        copy.put(key, cache);
        cacheMap.set(copy);
    }
    LOGGER.info("[{}] [subscribe] {}", agent.getName(), key);
    MetricsMonitor.getListenConfigCountMonitor().set(cacheMap.get().size());
    return cache;
}
```

该方法中一些逻辑已在注释中标出,这里就不在多说了.  
此处需要再进一步分析一下的是CacheData的构造方法的内部逻辑:
```
public CacheData(ConfigFilterChainManager configFilterChainManager, String name, String dataId, String group,
                 String tenant) {
    if (null == dataId || null == group) {
        throw new IllegalArgumentException("dataId=" + dataId + ", group=" + group);
    }
    this.name = name;
    this.configFilterChainManager = configFilterChainManager;
    this.dataId = dataId;
    this.group = group;
    this.tenant = tenant;
    listeners = new CopyOnWriteArrayList<ManagerListenerWrap>();
    this.isInitializing = true;
    this.content = loadCacheContentFromDiskLocal(name, dataId, group, tenant);
    this.md5 = getMd5String(content);
}
```

这里关注两个点:
*   this.content = loadCacheContentFromDiskLocal(name, dataId, group, tenant);  
    这里构造CacheData时,会首先尝试从本地存储的snapshot配置文件中尝试获取对用配置的内容
*   this.md5 = getMd5String(content);  
    对content做md5,用于后面对比配置是否被改变.


####    三.2   &emsp;&emsp;    cache.addListener

来看下内部源码
```
public void addListener(Listener listener) {
    if (null == listener) {
        throw new IllegalArgumentException("listener is null");
    }
    ManagerListenerWrap wrap = new ManagerListenerWrap(listener, md5);
    if (listeners.addIfAbsent(wrap)) {
        LOGGER.info("[{}] [add-listener] ok, tenant={}, dataId={}, group={}, cnt={}", name, tenant, dataId, group,
            listeners.size());
    }
}
```

这里,首先将我们希望对CacheData添加的监听器和前面构造CacheData时计算出的md5两个属性封装成ManagerListenerWrap对象.  
然后,将其添加到CacheData内部的listeners中.listeners是一个CopyOnWriteArrayList,这里使用了写时复制的方式来处理并发添加监听器的问题.



至此,nacos客户端获取配置的源码就分析完了.