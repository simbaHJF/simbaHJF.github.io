---
layout:     post
title:      "redis sentinel"
date:       2021-03-15 23:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - redis

---



## 导航
[一. 前言](#jump1)
<br>
[二. 集群节点信息状态](#jump2)
<br>
[三. 检测主观下线状态](#jump3)
<br>
[四. 检测客观下线状态](#jump4)
<br>
[五. 选举领头Sentinel](#jump5)
<br>
[六. 故障转移](#jump6)










<br><br>
## <span id="jump1">前言</span>

Sentinel(哨兵)是Redis的高可用解决方案:由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器,以及这些主服务器属下的所有从服务器,并在被监视的主服务器进入下线状态时,自动将下线主服务器属下的某个从服务器升级为新的主服务器,然后由新的主服务器代替已下线的主服务器继续处理命令请求.

[![6rUnjU.png](https://s3.ax1x.com/2021/03/15/6rUnjU.png)](https://imgtu.com/i/6rUnjU)
[![6rUY36.png](https://s3.ax1x.com/2021/03/15/6rUY36.png)](https://imgtu.com/i/6rUY36)
[![6rU4Ej.png](https://s3.ax1x.com/2021/03/15/6rU4Ej.png)](https://imgtu.com/i/6rU4Ej)
[![6rUI5n.png](https://s3.ax1x.com/2021/03/15/6rUI5n.png)](https://imgtu.com/i/6rUI5n)

当server1的下线时长超过用户设定的下线时长上限时,Sentinel系统就会对server1执行故障转移操作:
* 首先,Sentinel系统会挑选server1属下的的其中一个从服务器,并将这个被选中的从服务器升级为新的主服务器.
* 之后,Sentinel系统会向server1属下的所有从服务器发送新的复制指令,让它们成为新的主服务器的从服务器,当所有从服务器都开始复制新的主服务器时,故障转移操作执行完毕
* 另外,Sentinel还会继续监视已下线的server1,并在它重新上线时,将它设置为新的主服务器的从服务器



<br><br>
## <span id="jump2">二. 集群节点信息状态</span>

<br>
**<font size="4">创建网络连接</font>** <br>

Sentinel会向各个主节点及从节点创建两个网络连接
* 命令连接: 专门用于发送命令及接收回复
* 订阅连接: 订阅 \_sentinel\_:hello 频道

Sentinel各节点之间不会创建订阅订阅连接,只会创建命令连接.


<br>
**<font size="4">Sentinel获取主服务器信息</font>** <br>

Sentinel会以默认每10秒一次的频率通过命令连接向被监视的主服务器发送INFO命令,命令回复中包含两方面信息:
* 主服务器自身的信息
* 主服务器属下所有从服务器的信息


<br>
**<font size="4">Sentinel获取从服务器信息</font>** <br>

Sentinel以默认每10秒一次的频率通过命令连接向从服务器发送INFO命令,获取从服务器中中的相关信息
* 该从服务器复制的主服务器的相关信息
* 该从服务器自身的相关信息


<br>
**<font size="4">Sentinel订阅 \_sentinel\_:hello 频道</font>** <br>

当Sentinel与一个主或者从服务器建立起订阅连接之后,就会通过订阅连接向服务器发送以下命令<br>
``SUBSCRIBE _sentinel_:hello``<br>
也就是说,对于每个与Sentinel连接的服务器,Sentinel既通过命令连接向服务器的 \_sentinel\_:hello 频道发送信息,又通过订阅连接来通过服务器的 \_sentinel\_:hello 频道接收信息.<br>

对于监视同一个服务器的多个Sentinel来说,一个Sentinel发送的信息会被其他Sentinel接收到,这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知.<br>

因为一个Sentinel可以通过分析接收到的频道信息来获知其他Sentinel的存在,并通过发送频道信息来让其他Sentinel知道自己的存在,**<font color="red">所以用户在使用Sentinel的时候不需要提供各个Sentinel的地址信息,监视同一主服务器的多个Sentinel可以自动发现对方.</font>** <br>


<br>
**<font size="4">Sentinel向主服务器和从服务器发送信息</font>** <br>

默认情况下,Sentinel会以每两秒一次的频率,通过命令连接向所有被监视主服务器和从服务器的\_sentinel\相关命令以传递信息
[![6xvwND.png](https://z3.ax1x.com/2021/03/27/6xvwND.png)](https://imgtu.com/i/6xvwND)


<br><br>
## <span id="jump3">三. 检测主观下线状态</span>

默认情况下,Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例(包括主服务器、从服务器、其他sentinel在内)发送PING命令,并通过实例返回的PING命令回复来判断实例是否在线.<br>

如下例子:
* Sentinel1将向Sentinel2、主服务器master、从服务器slave1和slave2发送PING命令
* Sentinel2将向Sentinel1、主服务器master、从服务器slave1和slave2发送PING命令

实例对PING命令的回复可以分为以下两种情况:
* 有效回复:实例返回 +PONG、 -LOADING、 -MASTERDOWN三种回复的其中一种
* 无效回复:除有效回复外的其他回复,或者在指定时限内没有返回任何回复

Sentinel配置文件中的**<font color="red">down-after-milliseconds</font>**选项指定了Sentinel判断实例进入主观下线所需的时间长度:如果一个实例在down-after-milliseconds毫秒内,连续向Sentinel返回无效回复,那么Sentinel会修改这个实例所对应的实例结构,在结构的flags属性中打开SRI_S_DOWN标识,以此来表示这个实例已经进入主观下线状态.<br>

[![6sFqUA.png](https://s3.ax1x.com/2021/03/16/6sFqUA.png)](https://imgtu.com/i/6sFqUA)
[![6skpDg.png](https://s3.ax1x.com/2021/03/16/6skpDg.png)](https://imgtu.com/i/6skpDg)



<br><br>
## <span id="jump4">四. 检测客观下线状态</span>

当Sentinel将一个主服务器判断为主观下线之后,为了确认这个主服务器是否真的下线了,它会向同样监视这一主服务器的其他Sentinel进行询问,看它们是否也认为主服务器已经进入了下线状态(可以是主观下线或者客观下线).当Sentinel从其他Sentinel那里接收到足够数量的已下线判断之后,Sentinel就会将服务器判定为客观下线,并对主服务器执行故障转移操作.<br>


<br>
**<font size="4">发送 SENTINEL is-master-down-by-addr 命令</font>** <br>

Sentinel使用如下命令来询问其他Sentinel是否同意主服务器已下线<br>
``
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
``
[![6sA50e.png](https://s3.ax1x.com/2021/03/16/6sA50e.png)](https://imgtu.com/i/6sA50e)


<br>
**<font size="4">接收 SENTINEL is-master-down-by-addr 命令</font>** <br>

当一个Sentinel(目标Sentinel)接收到另一个Sentinel(源Sentinel)发来的"SENTINEL is-master-down-by"命令时,目标Sentinel会分析并取出命令请求中包含的各个参数,并根据其中的主服务器IP和端口号,检查主服务器是否已下线,然后向源Sentinel返回一条包含三个参数的Multi Bulk回复作为"SENTINEL is-master-down-by"命令的回复:
1. \<down_state\>
2. \<leader_runid\>
3. \<leader_epoch\>

[![6sER4s.png](https://s3.ax1x.com/2021/03/16/6sER4s.png)](https://imgtu.com/i/6sER4s)


<br>
**<font size="4">接收 SENTINEL is-master-down-by-addr 命令的回复</font>** <br>

根据其他Sentinel发回的对"SENTINEL is-master-down-by-addr"的命令回复,Sentinel将统计其他Sentinel同意主服务器已下线的数量,当这一数量达到配置指定的客观下线所需的数量时,Sentinel会将主服务器实例结构flags属性的SRI_O_DOWN标识打开,表示服务器已进入客观下线状态
[![6s1iOs.png](https://s3.ax1x.com/2021/03/16/6s1iOs.png)](https://imgtu.com/i/6s1iOs)
[![6s1Kl4.png](https://s3.ax1x.com/2021/03/16/6s1Kl4.png)](https://imgtu.com/i/6s1Kl4)



<br><br>
## <span id="jump5">五. 选举领头Sentinel</span>

当一个主服务器被判断为客观下线时,监视这个下线主服务器的各个Sentinel会进行协商,选举出一个领头Sentinel,并由领头Sentinel对下线主服务器执行故障转移操作.<br>

Redis选举领头Sentinel的规则和方法如下:
* 所有在线的Sentinel都有被选为领头Sentinel的资格,也即监视同一个主服务器的多个在线Sentinel中的任意一个都有可能成为领头Sentinel
* 每次进行领头Sentinel选举之后,不论选举是否成功,所有Sentinel的配置纪元的值都会自增一次.配置纪元实际上就是一个计数器,并没有什么特别的
* 在一个配置纪元里面,所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinle的机会,并且局部领头一旦设置,在这个配置纪元里面就不能再更改
* 每个发现主服务器进入客观下线的Sentinel都会要求其他Sentinel将自己设置为局部领头Sentinel
* Sentinel设置局部领头Sentinel的规则是先到先得
* 目标Sentinel在接收到"SENTINEL is-master-down-by-addr"命令之后,将向源Sentinel返回一条命令回复,回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元
* 源Sentinel在接收到目标Sentinel返回的命令回复之后,会检查恢复中leader_epoch参数的值和自己的配置纪元是否相同,如果相同的话,那么源Sentinel继续取出回复中的leader_runid参数,如果leader_runid参数的值和源Sentinel的运行ID一致,那么表示目标Sentinel将源Sentinel设置成了局部领头Sentinel
* 如果有某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel,那么这个Sentinel成为领头Sentinel.
* 因为领头Sentinel的产生需要半数以上Sentinel的支持,并且每个Sentinel在每个配置纪元里面只能设置一次局部领头Sentinel,所以在一个配置纪元里面,只会出现一个领头Sentinel.
* 如果在给定时限内,没有一个Sentinel被选举为领头Sentinel,那么各个Sentinel将在一段时间之后再次进行选举,直到选出领头Sentinel为止



<br><br>
## <span id="jump6">六. 故障转移</span>

在选举产生出领头Sentinel之后,领头Sentinel将对已下线的主服务器执行故障转移操作,该操作包含以下三个步骤:
1. 在已下线主服务器属下的所有从服务器里面,挑选出一个从服务器,并将其转换为主服务器
2. 让已下线主服务器属下的所有从服务器复制新的主服务器
3. 将已下线主服务器设置为新的主服务器的从服务器,当这个旧的主服务器重新上线时,它就会成为新的主服务器的从服务器