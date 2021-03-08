---
layout:     post
title:      "synchronized锁优化"
date:       2021-03-06 17:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - JVM

---



## 导航
[一. HotSpot虚拟机对象头Mark Word](#jump1)
<br>
[二. 重量级锁](#jump2)
<br>
[三. 轻量级锁](#jump3)
<br>
[四. 偏向锁](#jump4)
<br>




<br><br>
## <span id="jump1">一. HotSpot虚拟机对象头Mark Word</span>

由于对象头信息是与对象自身定义的数据无关的额外存储成本,考虑到Java虚拟机的空间使用效率,Mark Word被设计成一个非固定的动态数据结构,以便在极小的空间内存储尽量多的信息.它会根据对象的状态复用自己的存储空间.

[![6uZQJO.png](https://s3.ax1x.com/2021/03/06/6uZQJO.png)](https://imgtu.com/i/6uZQJO)



<br><br>
## <span id="jump2">二. 重量级锁synchronized</span>

<br>
**<font size="4">synchronized作用</font>** <br>

* 原子性: synchronized保证语句块内操作是原子的
* 可见性: synchronized保证可见性(通过"退出同步块之前,必须先把此变量同步回主内存"来保证)
* 有序性: synchronized保证有序性(通过"同一时间,只能有一个线程在执行同步块逻辑"来保证)


<br>
**<font size="4">synchronized的使用</font>** <br>

* 修饰实例方法,对当前实例对象加锁
* 修饰静态方法,多当前类的Class对象加锁
* 修饰代码块,对synchronized括号内的对象加锁


<br>
**<font size="4">synchronized的实现原理</font>** <br>

<font color="red">jvm基于进入和退出Monitor对象来实现方法同步和代码块同步</font> <br>

方法级的同步是隐式的,即无需通过字节码指令来控制的,它实现在方法调用和返回操作之中.JVM可以从方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法.当方法调用时,调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置,如果设置了,执行线程将先持有monitor(虚拟机规范中用的是管程一词), 然后再执行方法,最后在方法完成(无论是正常完成还是非正常完成)时释放monitor.<br>

代码块的同步是利用monitorenter和monitorexit这两个字节码指令.它们分别位于同步代码块的开始和结束位置.当jvm执行到monitorenter指令时,当前线程试图获取monitor对象的所有权,如果未加锁或者已经被当前线程所持有,就把锁的计数器+1;当执行monitorexit指令时,锁计数器-1;当锁计数器为0时,该锁就被释放了.如果获取monitor对象失败,该线程则会进入阻塞状态,直到其他线程释放锁.<br>

关于ACC_SYNCHRONIZED 、monitorenter、monitorexit指令,可以看一下下面的反编译代码
```
public class SynchronizedDemo {
    public synchronized void f(){    //这个是同步方法
        System.out.println("Hello world");
    }
    public void g(){
        synchronized (this){		//这个是同步代码块
            System.out.println("Hello world");
        }
    }
    public static void main(String[] args) {

    }
}
```

使用javap -verbose SynchronizedDemo反编译后得到
[![6uu5U1.jpg](https://s3.ax1x.com/2021/03/06/6uu5U1.jpg)](https://imgtu.com/i/6uu5U1)
[![6uKkrQ.jpg](https://s3.ax1x.com/2021/03/06/6uKkrQ.jpg)](https://imgtu.com/i/6uKkrQ)

对于同步方法,反编译后得到ACC_SYNCHRONIZED 标志,对于同步代码块反编译后得到monitorenter和monitorexit指令.<br>

Synchronized是通过对象内部的一个叫做监视器锁(monitor)来实现的,监视器锁本质又是依赖于底层的操作系统的Mutex Lock(互斥锁)来实现的,需要由用户态切换到内核态,这个成本非常高,状态之间的转换需要相对比较长的时间,这就是为什么Synchronized效率低的原因.因此,这种依赖于操作系统Mutex Lock所实现的锁我们称之为"重量级锁".<br>

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗,引入了"偏向锁"和"轻量级锁": 至此,锁一共有4种状态,级别从低到高依次是:无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态.锁可以升级但不能降级.举例来讲,原本某个锁是偏向锁状态,一旦由于竞争的出现,锁升级为轻量级锁或者是重量级锁,它就不能再降级回低级别锁了,会一直保持在高级别锁状态.<br>



<br><br>
## <span id="jump3">三. 轻量级锁</span>

轻量级锁并不是用来代替重量级锁的,它设计的初衷是在没有多线程竞争的前提下,减少传统的重量级锁使用操作系统互斥量产生的性能消耗.<br>


**<font size="4">加锁过程</font>** <br>

在代码即将进入同步块的时候,如果此时同步对象没有被锁定("锁标志位01"状态),虚拟机首先将在当前线程的栈帧中建立一个名为锁记录(Lock Record)的空间,用于存储锁对象目前的 Mark Work 拷贝(官方为这份拷贝加了一个Displaced前缀,即 Displaced Mark Word),这时候线程堆栈与对象头的状态如下图所示.<br>
[![6ulWz6.png](https://s3.ax1x.com/2021/03/06/6ulWz6.png)](https://imgtu.com/i/6ulWz6)

然后虚拟机将使用CAS操作尝试把对象的Mark Word更新为指向Lock Record的指针.如果这个更新动作成功了,即代表该线程拥有了这个对象的锁,并且对象Mark Word的锁标志位将转变为"00",表示此对象处于轻量级锁定状态.此时线程堆栈与对象头的状态如下图所示.<br>
[![6u1FWq.png](https://s3.ax1x.com/2021/03/06/6u1FWq.png)](https://imgtu.com/i/6u1FWq)

如果这个更新操作失败了,那就意味着至少存在一条线程与当前线程竞争获取该对象的锁.虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧,如果是,说明当前线程已经拥有了这个对象的锁,那直接进入同步块继续执行就可以了,否则就说明这个锁对象已经被其他线程抢占了.如果出现两条以上的线程争用同一个锁的情况,那轻量级锁就不再有效,必须要膨胀为重量级锁,锁标志的状态值变为"01",此时Mark Word中存储的就是指向重量级锁(monitor,互斥量,操作系统内核级别)的指针后面等待锁的线程也必须进入阻塞状态.<br>


**<font size="4">解锁过程</font>** <br>

解锁过程同样是通过CAS操作来进行的,如果对象的Mark Word仍然指向线程的锁记录,那就用CAS操作把对象当前的Mark Word和线程中复制的Displaced Mark Word替换回来.假如能够成功替换,那整个同步过程就顺利完成了;如果替换失败了,则说明有其他线程尝试过获取该锁(此时,该锁应该已经膨胀为重量级锁了),就要在释放锁的同时,唤醒被挂起的线程.<br>


**<font size="4">小结</font>** <br>

轻量级锁能提升程序同步性能的依据是"对于绝大部分的锁,在整个同步周期内都是不存在竞争的"这一经验法则.如果没有竞争,轻量级锁便通过CAS操作成功避免了使用互斥量的开销;但如果确实存在锁竞争,除了互斥量本身的开销外,还额外发生了CAS操作的开销.因此在有竞争的情况下,轻量级锁反而会比传统的重量级锁更慢.<br>



<br><br>
## <span id="jump4">四. 偏向锁</span>

偏向锁也是JDK6中引入的一项锁优化措施,它的目的是消除数据在无竞争情况下的同步原语,进一步提高程序的性能.如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量,那偏向锁就是在无竞争的情况下把整个同步都消除掉,连CAS操作都不去做了.<br>

偏向锁会偏向于第一个获得它的线程,如果在接下来的执行过程中,该锁一直没有被其他的线程获取,则持有偏向锁的线程将永远不需要再进行同步.<br>

当锁对象第一次被线程获取的时候,虚拟机会把对象头中的标志位设置为"01",把偏向模式设置为"1",表示进入偏向模式.同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中.如果CAS操作成功,持有偏向锁的线程以后每次进入这个锁相关的同步块时,虚拟机都可以不再进行任何同步操作(例如加锁、解锁及对Mark Word的更新操作等).<br>

一旦出现另外一个线程尝试获取这个锁的情况,偏向模式就马上宣告结束.根据锁对象目前是否处于被锁定的状态决定是否撤销偏向(偏向模式设置为"0"),撤销后标志位恢复到未锁定(标志位为"01")或轻量级锁定(标志位为"00")的状态,后续的同步操作就按照轻量级锁那样去执行.偏向锁、轻量级锁的状态转化及对象Mark Word的关系如下图

[![6uBgxg.png](https://s3.ax1x.com/2021/03/06/6uBgxg.png)](https://imgtu.com/i/6uBgxg)

偏向锁也是一个带有效益权衡(Trade Off)性质的优化,如果程序中大多数的锁都总是被多个不同的线程访问,那偏向模式就是多余的.在具体问题具体分析的前提下,有时候使用参数-XX:UseBiasedLocking来禁止偏向锁优化反而可以提升性能.<br>