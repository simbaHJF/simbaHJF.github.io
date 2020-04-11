---
layout:     post
title:      "分布式系统--Raft总结"
date:       2020-04-10 20:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 分布式

---



#	Raft基本

一个Raft集群包含多个server节点,通常是5,这样一来,能够容错两台server的宕机情况.

在任意时间点,server节点必定处于三种状态之一:

*	leader  
	处理所有的client请求(如果一个client联络的是一个follower节点,那么follower节点会将它重定向到leader)

*	follower  
	自身不发出任何request请求,只是简单的响应来自leader或者candidate的request请求.

*	candidate  
	用于选举一个新的leader

正常运行状态下,只存在一个leader节点,其他节点全是follower状态.

<br>

Raft将时间划分为一个个的term,划分term的时间长短是任意的.term以连续的整数来标记.每个term以选举开始.如果一个candidate赢得了选举,那么它将作为leader,在这轮term的其余时间内提供服务.在一些情况下,一次选举会得到分裂投票的结果,在这种情况下,这轮term将会在没有leader的情况下结束,并且一轮新的term(伴随着一次新的选举)将会立即开始.Raft保证在一个给定term中,最多有一个leader.

在Raft中,term扮演逻辑时钟的角色,通过term来使server节点发现果实信息,比如过时的leader.每个server节点都会存储当前term号,这个term是随时间单调递增的.当server间通信的时候,会交换各自的当前term信息,如果一个server的当前term号小于其他server的,那么这个term号小的server会将其term号更新为比他大的那个term号.如果一个candidate或leader发现自己的term过时了,它会立即将自己的状态恢复为follower状态.如果一个server收到的请求中携带的term信息是过时的,server将拒绝这个请求.

Raft的server之间通信使用的是远程过程调用(RPCs),在这个基础共识算法中,只有两种RPC.
*	RequestVote Rpcs----在选举过程中,被candidate创建和发出
*	AppendEntries Rpcs----1.在日志entry复制时,被leader创建和发出;2.心跳形式发出

如果server节点没有及时收到响应,则会重试RPC,并且server节点时并行发出RPC请求的,以此来获得最佳性能.

<br><br>


#	Leader选举

Raft采用心跳机制来触发leader选举.当server启动的时候,他们最初是follower状态.只要一个server能持续收到来自leader或者是candidate的合法RPCs,那么它就保持follower状态.

leader周期性地给所有follower发送心跳(AppendEntries RPCs,但是不携带log entry)来保持它的权威.如果一个follower超过一个时间周期没有收到通信信息,它就会假设此时没有可用的leader,并且它会开启一次选举来选出一个新的leader,这个时间周期称为:election timeout(让我翻译的话,选举超时?)

为了开始一次选举,follower会增加它的term号并且转变为candidate状态.之后,它会给自己投一票并且并行的向集群中的其他server节点发出RequestVote RPCs请求.

一个candidate将继续保持这个candidate状态直到发生下面三件事中的其中任意一件:
*	这个candidate它自己赢得了此次选举
*	其他server将自己建立为leader
*	一个周期走完但是没有赢得选举.

下面对上述三种情况分别讨论:

第一种情况:一个candidate如果获得了集群中多数server的投票,那么它将赢得此次选举.在一个term中,每个server节点最多能给一个candidate投票,投票遵循先到先得的基础原则.另外,多数规则保证了在一个特定的term周期中,最多只有一个candidate能够赢得选举.一旦一个candidate赢得了选举,他就称为leader,然后他会给所有server节点发送心跳消息,一是用来建立权威,而是用来组织新的选举.

第二种情况:在等待投票过程中,一个candidate可能会收到AppendEntries RPC,这个RPC来自另外一个声明自己为leader的server节点.如果这个leader节点的term(被包括在RPC中)是大于等于自己这个candidate节点的,那么这个candidate节点将承认发来RPC消息的那个leader的合法性,并且自身会回到follow状态.如果RPC中的term小于candidate节点自己当前的term,那么这个candidate会拒绝这个RPC并且继续保持candidate状态.

第三种情况:一个candidate节点姐妹有赢得选举,也没有失去选举资格:例如同时有许多follower同时成为了candidate,那么投票将被分裂,导致没有一个candidate获得多数节点的投票.这时,每一个candidate将会超时,然后它会增加team号,初始化新一轮RequestVote RPC,以此来开启一个新的选举.然而这里如果不加额外措施的话,投票分裂可能会无休止的重复发生.

Raft采用随机election timeout来保证分裂投票的情况极少出现并且能被快速解决.  
*	首先,为了避免分裂投票,election timeout是从固定间隔中随机选择的(例如,150-300ms).这会把server分散开,从而在大多数情况下,只有一个server会超时.它会在其他节点超时之前,赢得选举并发送心跳.
*	每个candidate在选举开始时重置它的随机election timeout,并且它会等到这个超时时间之后才会开启下次选举,这就减少了在新选举中再次发生分裂表决的可能性.


节点状态转化图:
![GHRNad.png](https://s1.ax1x.com/2020/04/11/GHRNad.png)

term示意图:
![GHWiod.png](https://s1.ax1x.com/2020/04/11/GHWiod.png)


#	日志复制

