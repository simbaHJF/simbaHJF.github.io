---
layout:     post
title:      "rocketMQ broker端位移存储"
date:       2021-06-02 18:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - MQ

---





Broker端使用一个map在内存中缓存位移数据
```
private ConcurrentMap<String/* topic@group */, ConcurrentMap<Integer, Long>> offsetTable =
        new ConcurrentHashMap<String, ConcurrentMap<Integer, Long>>(512);
```

Broker端接收Consumer端提交的位移数据的处理逻辑在ConsumerManageProcessor中,最终底层会更新ConsumerOffsetManager中的上述offsetTable源码如下:
```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.GET_CONSUMER_LIST_BY_GROUP:
            return this.getConsumerListByGroup(ctx, request);
        case RequestCode.UPDATE_CONSUMER_OFFSET:
            return this.updateConsumerOffset(ctx, request);
        case RequestCode.QUERY_CONSUMER_OFFSET:
            return this.queryConsumerOffset(ctx, request);
        default:
            break;
    }
    return null;
}


private RemotingCommand updateConsumerOffset(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    final RemotingCommand response =
        RemotingCommand.createResponseCommand(UpdateConsumerOffsetResponseHeader.class);
    final UpdateConsumerOffsetRequestHeader requestHeader =
        (UpdateConsumerOffsetRequestHeader) request
            .decodeCommandCustomHeader(UpdateConsumerOffsetRequestHeader.class);
    this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), requestHeader.getConsumerGroup(),
        requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}


public void commitOffset(final String clientHost, final String group, final String topic, final int queueId,
    final long offset) {
    // topic@group
    String key = topic + TOPIC_GROUP_SEPARATOR + group;
    this.commitOffset(clientHost, key, queueId, offset);
}


private void commitOffset(final String clientHost, final String key, final int queueId, final long offset) {
    ConcurrentMap<Integer, Long> map = this.offsetTable.get(key);
    if (null == map) {
        map = new ConcurrentHashMap<Integer, Long>(32);
        map.put(queueId, offset);
        this.offsetTable.put(key, map);
    } else {
        Long storeOffset = map.put(queueId, offset);
        if (storeOffset != null && offset < storeOffset) {
            log.warn("[NOTIFYME]update consumer offset less than store. clientHost={}, key={}, queueId={}, requestOffset={}, storeOffset={}", clientHost, key, queueId, offset, storeOffset);
        }
    }
}
```



BrokerController启动时会启动定时任务,默认每隔5秒将位移数据持久化到磁盘
```
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        try {
            BrokerController.this.consumerOffsetManager.persist();
        } catch (Throwable e) {
            log.error("schedule persist consumerOffset error.", e);
        }
    }
}, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
```