---
layout:     post
title:      "分布式锁"
date:       2021-04-16 15:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 分布式

---







## 导航
[一. 分布式锁--基于Redis](#jump1)
<br>
[二. 分布式锁--基于Zookeeper](#jump2)
<br>













<br><br>
## <span id="jump1">一. 分布式锁--基于Redis</span>

这里主要分析开源框架Redisson中实现的Redis分布式锁的底层原理.<br>

通过Redisson的api进行加锁解锁的代码示例如下:
```
public static void main(String[] args) {

    Config config = new Config();
    config.useSingleServer().setAddress("redis://127.0.0.1:6379");
    config.useSingleServer().setPassword("redis1234");
    
    final RedissonClient client = Redisson.create(config);  
    RLock lock = client.getLock("lock1");
    
    try{
        lock.lock();
    }finally{
        lock.unlock();
    }
}
```


<br>
**<font size="4">加锁流程</font>** <br>

现假设某个客户端要加锁.如果该客户端面对的是一个redis cluster集群,它首先会根据hash节点选择一台机器,这里注意,仅仅是选择一台机器,这点很关键!<br>

方法底层会调用请求Redis执行一段Lua脚本:
```
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit,     
                            long threadId, RedisStrictCommand<T> command) {

        //过期时间
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  //如果锁不存在，则通过hset设置它的值，并设置过期时间
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  //如果锁已存在，并且锁的是当前线程，则通过hincrby给数值递增1
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  //如果锁已存在，但并非本线程，则返回过期时间ttl
                  "return redis.call('pttl', KEYS[1]);",
        Collections.<Object>singletonList(getName()), 
                internalLockLeaseTime, getLockName(threadId));
    }
```

总结加锁流程如下图,同时在该流程中体现了互斥锁以及可重入锁:
[![cfKAU0.md.png](https://z3.ax1x.com/2021/04/16/cfKAU0.md.png)](https://imgtu.com/i/cfKAU0)


<br>
**<font size="4">watch dog自动延期机制</font>** <br>

客户端加锁的锁key默认生存时间是30秒,如果超过了30秒,该持有锁的客户端没操作完,还想持有这把锁,该怎么办呢?<br>

这里,Redisson有一个watch dog机制.客户端一旦加锁成功,就会启动一个watch dog看门狗,它是一个后台线程,会每隔10秒检查一下,如果客户端还持有锁key,那么就会延长锁key的生存时间.<br>


<br>
**<font size="4">解锁流程</font>** <br>

解锁逻辑会调用到如下底层方法,同样会请求Redis执行Lua代码段:
```
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, EVAL,
    
            //如果锁已经不存在， 发布锁释放的消息
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            //如果释放锁的线程和已存在锁的线程不是同一个线程，返回null
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                "return nil;" +
            "end; " +
            //通过hincrby递减1的方式，释放一次锁
            //若剩余次数大于0 ，则刷新过期时间
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            "if (counter > 0) then " +
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "return 0; " +
            //否则证明锁已经释放，删除key并发布锁释放的消息
            "else " +
                "redis.call('del', KEYS[1]); " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; "+
            "end; " +
            "return nil;",
    Arrays.<Object>asList(getName(), getChannelName()), 
        LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

}
```

总结解锁流程如下图:
[![cfFOAg.md.png](https://z3.ax1x.com/2021/04/16/cfFOAg.md.png)](https://imgtu.com/i/cfFOAg)


<br>
**<font size="4">Red Lock算法</font>** <br>

以上均是针对单节点的加锁解锁机制,但考虑这样场景:<br>

加锁解锁节点为 master-slave 架构,锁被持有的过程中,master节点宕机了,而 master - slave 架构下,是异步复制的,此时slave还没将相关数据复制过来,此时进行故障转移选主,slave节点被提升为master,此时另外一个client向该新当选的master(由slave提升来的)发起加锁,这时会成功,就造成锁一致性的破坏.<br>

为了解决这个问题,因此有了 Red Lock 算法的出现.<br>

Red Lock的基本思想就是,在前面分析的单机加锁解锁基础上,对master-slave架构下的集群节点进行过半加锁机制,以保证分布式master-slave架构下,即便master节点宕机,也会有锁一致性保证.<br>

<font color="red">核心即: 成功加锁的实例数>= N/2 + 1,才认为加锁成功.</font>


<br>
**<font size="4">高并发下,如大促库存减扣场景下,Redis的锁优化</font>** <br>

<font color="red">核心即: 锁分段技术,参见ConcurrentHashMap的思想及LongAdder的思想</font>



<br><br>
## <span id="jump2">二. 分布式锁--基于Zookeeper</span>

<font color="red">核心:zk的临时顺序节点</font> <br>

[![cfyJFP.png](https://z3.ax1x.com/2021/04/16/cfyJFP.png)](https://imgtu.com/i/cfyJFP)

这里注意红框中,判断自身不是排第一的临时顺序节点,为什么还要对迁移节点添加监听呢?<br>

这是为了应对这种场景: 在zk的my_lock节点下建立了一些列临时顺序节点后,加锁正常运行中,这时突然排第三的节点机器宕机了,这时为了确保原来排第四的临时顺序节点在后续能成功加到锁,必须修正其监听为原先排第二的节点,因为随着节点三的宕机,监听链已经断了,需要修复调整.<br>

<font color="red">另外,zk锁这里不涉及到前面Redis分布式锁中主节点宕机的问题.因为ZK实现的是天然的过半写成功才给客户端返回写入成功的一致性协议,本身具有CP性,因此不存在这个问题.</font> <br>

<font color="red">另外再提一句,ZK并不适合高并发的写入的场景,因为其ZAB协议的本身机制,高并发写更新时,性能很差</font> <br>

