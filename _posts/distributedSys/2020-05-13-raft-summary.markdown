---
layout:     post
title:      "分布式系统--Raft总结"
date:       2020-05-13 20:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 分布式

---



#	5.1 Raft基本

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

Raft属性:
![GbSLR0.png](https://s1.ax1x.com/2020/04/11/GbSLR0.png)

<br><br>


#	5.2 Leader选举

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

<br><br>


#	5.3 日志复制

一旦leader被选定了,它就开始为client请求服务.每一个client请求包含一个将被复制状态机运行的command.leader将这个command作为一个新的entry追加到自己的log中,然后并行向其他每一个server节点发出AppendEntries RPCs来执行entry复制.当entry已经被安全复制了,leader会将这个entry应用到它自己的状态机中,然后向client返回执行结果.如果followers宕机或者运行缓慢,或者发生网络丢包,leader会无限重试AppendEntries RPCs,直到所有的followers都最终存储了所有的log entry.

log示意图:
![GHIiG9.png](https://s1.ax1x.com/2020/04/11/GHIiG9.png)

每一个log entry都会存储一个状态机command和term.log entry中的term号用于发现不同log之间的不一致性和确保其他一些特性.每一个log entry还有一个整数型的索引用于确认其在log中的位置.

leader会决定什么时候将一个log entry应用到状态机中是安全的,这样的entry被叫做committed.Raft保证committed entries是持久化的并且最终会被所有的可用状态机执行.当创建这个entry的leader已经将它复制到多数server中,这时,这个log entry就被提交了(committed).这还将提交leader日志中的所有先前entry,包括先前领导者创建的entry.leader记录被提交的最高索引号,并在以后的AppendEntries RPC(包括heartbeats)中包含该索引,以便其他server获取.一旦一个follower了解到一个log entry被提交了,它就会将该log entry应用到它的本地状态机(按日志顺序).

设计Raft日志机制是为了保持不同服务器上日志之间的高度一致性.这不仅简化了系统行为,保证了它更加可预测,而且也是确保安全性的重要组成部分.Raft维护如下两个特性,它们一起构成了Raft的Raft中的Log Matching Property(日志匹配属性,Figure 3):
*	如果不同日志中的两个entry有相同的index和term,那么他们存储着相同的command.
*	如果不同日志中的两个entry有相同的index和term,他们这些不同的日志在该entry之前的所有entry是完全相同的.

第一个属性源于这样一个事实:一个leader在给定log索引和term下,最多只会创建一个entry,并且这个log entry永远不会更改它们在日志中的位置.  
第二个属性是由AppendEntries所执行的一个一致性检查来保证的.当发送一个AppendEntries Rpc的时候,leader将log中紧邻此entry的上一个entry的index和term包含在这个RPC请求中,如果follower在自己的log没有找到与RPC请求中包含的index和term有着相同信息的entry,那么follower会拒绝这个新的entry.

这个一致性检查是通过归纳法生效的:log的初始空状态是满足Log Matching Property的,然后这个一致性检查在log扩展的时候来保持Log Matching Property.因此,只要AppendEntries返回成功,leader就会知道:follower中,截止到这个最新发送过去的entry之前,其log与自己的是完全相同的.

在正常运行过程中,leader和followers的log是保持一致的,所以AppendEntries的一致性检查一直都不会失败.但是,leader宕机会导致不一致性(老的leader还没有将其日志中的所有entry都做完复制).这些不一致可能会导致一系列的leader和follower崩溃.

下图描述了一些导致followers和leader不一致的情况:
![GbMmgU.png](https://s1.ax1x.com/2020/04/11/GbMmgU.png)

follower可能会缺少leader中出现的entry,可能会包含leader中没有出现的entry,或者前面两种情况都有.日志中缺少的和无关的条目可能跨越多个term.

在Raft中,leader强制follower复制leader的log来解决不一致问题.这意味着follower中的冲突entry将会被leader的log中entry所覆盖.在后面的5.4部分,会说明:在此基础之上,再增加一些限制,就能保证这种操作是安全的.

leader为了让follower的log变为与自己的一致,leader必须找到与follower中的log保持一致的最后一个entry,记为一个点,然后删除follower的log中这个点之后的所有entry,然后将leader自己的log中这个点之后的所有entry发送给follower.所有的这些行为都是发生在AppendEntries RPCs所执行的一致性检查的响应中.leader会为每一个follower维护一个nextIndex,这个索引指向leader将要向某个follower发送的下一个log entry.当一个leader第一次掌权时,它会初始化所有的nextIndex为该leader当前log中最后一个entry的下一个.如果某个follower的log与leader的不一致,AppendEntries的一致性检查机制会使下一次AppendEntries RPC失败.在拒绝之后,leader减小nextIndex,然后重试AppendEntries RPC.最终nextIndex将会到达一个一个点,在这个点上,leader和follower的log是匹配的.此时,AppendEntries就会成功了,这样就移除了follower的log中有冲突的entry并且向log中追加写入了leader中的log entry.一旦AppendEntries成功了,follower的log就与leader中的一致了,并且它会在该term的余下时间内保持这种一致性.

如果需要,可以对这个协议进行优化以减少AppendEntries RPCs被拒绝的次数.例如,当拒绝一个AppendEntries请求的时候,follower可以反馈冲突entry的term和该term下它存储的第一个entry.通过这个信息,leader能够以绕开该term下的所有冲突entry的方式来减小nextIndex值;对于冲突的这些entry,每一个term只需要一次AppendEntries RPCs,而不是每个冲突entry一次AppendEntries RPCs.在实践中,我们怀疑这种优化是否有必要,因为失败很少发生并且冲突entry不会有很多.

通过这个机制,当一个leader掌权时,它不需要为恢复日志一致性做任何特殊动作.它只是开始正常运转,log会通过响应AppendEntries的一致性检查机制而自动完成聚合.一个leader是绝对不会重写或者是删除自己log中的entry的(Raft属性中的Leader Append-Only Property).

<br><br>


#	5.4	安全性

在前文讨论的规则基础上,追加约束.以保证安全性

###	5.4.1	选举限制

所有的基于leader的共识算法要求leader最终存储了所有提交entry.一些共识算法是通过一些附加机制来实现这一点的,但是这也引入了需要额外考虑的复杂性.Raft采用一个简单的方式保证所有的commited entry都会存储在新leader中,而且它不是靠向leader传输entry来完成的.这意味着,在Raft中log entry只有一个流向,就是从leader到follower,leader绝不会覆写已存在于log中的entry.

Raft借助投票进程来实现这一点,如果一个candidate没有包含所有committed entry,那么它会被阻止称为leader.candidate为了被选举,它必须与集群中的多数节点联系,这意味着,**<font color="red">每一个committed entry一定会出现在这个多数节点中的某一个里.</font>**,如果一个candidate的log与多数节点中每一个的log相比,都是至少不落后与它的,这样就能保证该candidate持有了所有的committed entry.RequestVote RPC实现了这一约束:在RPC中包含了candidate的log信息,如果follower发现自己的log比candidate的log要新,那么就拒绝给它投票.

Raft通过比较最后一个entry的index和term来判断哪个log更新.如果两个log之间的最后一个entry有着不同的term,那么term大的更新.如果log之间最后一个entry有相同的term,那么谁的log更长,谁就更新.

###	5.4.2	从先前的term提交entry

如5.3中描述的,当一个entry被多数节点存储了,那么它可以被提交.如果leader在他提交entry之前宕机,下任leader将尝试完成这个entry的复制.但是,leader不能立即得出如下结论:一旦上一个term中的某个entry被多数节点存储了,就认为它已被提交.
下图Figure-8说明了这样一种情况:一个老的entry被多数节点存储了,但是仍然被下任leader所覆写.
![Jp3bBd.png](https://s1.ax1x.com/2020/04/14/Jp3bBd.png)

为了消除Figure-8中的类似问题,Raft从不通过计数副本数来提交先前term的log entry.只有来自于当前leader,当前term下的log entry才通过计数副本数来提交;一旦以这种方式提交了来自当前term的entry,由于Log Matching Property,将间接提交所有先前的entry.这里有一些情况下,leader能够安全的得出结论:一个老的log entry已经被提交了(例如,这个entry已经被每一个节点存储了),但是,Raft仍然使用这种更为保守方式来保持简单性(**<font color="red">指的是Log Matching Property,以及前文所讲的日志复制,冲突时leader强制follower进行log entry的overwrite以使得和leader保持一致.也就是说,Figure-8
中的c,S1将不会再复制term2给S3,另外,即使在b阶段S1将term复制给了S2,S3,但是在提交前宕机了,到d时,index2的位置还是会被term3覆写.</font>**).

Raft引入了这种复杂性的原因是:当一个leader复制先前term下的entry时,log entry将保留其原有term编号.在其他的共识算法中,如果一个新的leader复制先前term下的entry,它会用一个新的term号来标记这个entry.Raft的方法使得log entry的推理变得容易,因为它的entry的term保持不变.此外,新的leader发送的先前term下的entry很少(其他算法在重新提交这些entry之前,必须发送冗余的log entry来对他们进行重编号)