---
layout:     post
title:      "hystrix 基本原理"
date:       2021-04-26 14:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - hystrix

---







## 导航
[一. 容错限流的需求](#jump1)
<br>
[二. 容错限流的原理](#jump2)
<br>
[三. Hystrix的功能场景](#jump3)
<br>
[四. Hystrix核心功能](#jump4)
<br>











<br><br>
## <span id="jump1">一. 容错限流的需求</span>

在复杂的分布式系统中通常有很多依赖,如果一个应用不能对来自依赖的故障进行隔离,那么应用本身就处于被拖垮的风险中.在一个高流量的网站中,某一个单一后端一旦发生延迟,将会在数秒内导致所有的应用资源被耗尽,这也就是我们常说的雪崩效应.<br>

比如在电商系统的下单业务中,在订单服务创建订单后同步调用库存服务进行库存的扣减,假如库存服务出现了故障,那么会导致下单请求线程会被阻塞,当有大量的下单请求时,则会占满应用连接数从而导致订单服务无法对外提供服务.<br>



<br><br>
## <span id="jump2">二. 容错限流的原理</span>

对于基本的容错限流模式,主要有以下几点需要考量:
* 主动超时:在调用依赖时尽快的超时,可以设置比较短的超时时间,比如2s,防止长时间的等待
* 限流:限制最大并发数
* 熔断:错误数达到阈值时,类似于保险丝熔断
* 隔离:隔离不同的依赖调用
* 服务降级:资源不足时进行服务降级


<br>
**<font size="4">断路器模式</font>** <br>

[![gSBLkR.png](https://z3.ax1x.com/2021/04/26/gSBLkR.png)](https://imgtu.com/i/gSBLkR)

实现流程为:当断路器的开关为关闭时(对应图中的绿色),每次请求进来都是成功的,当后端服务出现问题,请求出现的错误数达到一定的阈值,并且出错的次数也达到设置的出错阈值,则会触发断路器为打开状态(对应图中的红色),在断路器为打开状态时,进来的所有请求都会被拒绝,当然也不是一直会拒绝请求,而是弹性的,过了特定的时间后,断路器会进入半打开状态(对应图中的黄色),这时会让一部分请求通过进行尝试,如果尝试还是有问题,则继续进入打开状态,如果尝试没有问题了,则会进入关闭状态.<br>

三种状态:
* open状态,说明打开熔断,也就是服务调用方执行本地降级策略,不进行远程调用
* closed状态,说明关闭了熔断,这时候服务调用方直接发起远程调用
* half-open状态,则是一个中间状态,当熔断器处于这种状态时候,直接发起远程调用

三种状态转换:
* closed -> open
	> 正常情况下熔断器为closed状态,当访问同一个接口次数超过设定阈值并且错误比例超过设置错误阈值时候,就会打开熔断机制,这时候熔断器状态从closed变为open
* open -> half-open
	> 当服务接口对应的熔断器状态为open状态时候,所有服务调用方调用该服务方法时候都是执行本地降级方法,那么什么时候才会恢复到远程调用那?Hystrix提供了一种测试策略,也就是设置了一个时间窗口,从熔断器状态变为open状态开始的一个时间窗口内,调用该服务接口时候都委托服务降级方法进行执行.如果时间超过了时间窗口,则把熔断状态从open->half-open,这时候服务调用方调用服务接口时候,就可以发起远程调用而不再使用本地降级接口,如果发起远程调用还是失败,则重新设置熔断器状态为open状态,从新记录时间窗口开始时间
* half-open -> closed
	> 当熔断器状态为half-open,这时候服务调用方调用服务接口时候,就可以发起远程调用而不再使用本地降级接口,如果发起远程调用成功,则重新设置熔断器状态为closed状态


<br>
**<font size="4">舱壁隔离模式</font>** <br>

舱壁隔离模式可以对资源进行隔离,类似于船的船舱都是被隔离开来的,当其中一个或者几个船舱出现问题,比如漏水,是不会影响到其他的船舱的,从而实现一种资源隔离的效果.<br>



<br><br>
## <span id="jump3">三. Hystrix的功能场景</span>

* 资源隔离: 包括线程池隔离和信号量隔离,避免某个依赖出现问题会影响到其他依赖.
* 断路器: 当请求失败率达到一定的阈值时,会打开断路器开关,直接拒绝后续的请求,并且具有弹性机制,在后端服务恢复后,会自动关闭断路器开关.
* 降级回退:当断路器开关被打开,服务调用超时/异常,或者资源不足(线程、信号量)会进入指定的fallback降级方法.
* 请求结果缓存: hystrix实现了一个内部缓存机制,可以将请求结果进行缓存,那么对于相同的请求则会直接走缓存而不用请求后端服务,<font color="red">使用场景很少,不是重点</font>
* 请求合并: 可以实现将一段时间内的请求合并,然后只对后端服务发送一次请求.<font color="red">使用场景很少,不是重点</font>



<br><br>
## <span id="jump4">四. Hystrix核心功能</span>

<br>
**<font size="4">资源隔离</font>** <br>

资源隔离的思想参考舱壁隔离模式,在hystrix中提供了两种资源隔离策略:线程池隔离、信号量隔离
[![gScb7D.png](https://z3.ax1x.com/2021/04/26/gScb7D.png)](https://imgtu.com/i/gScb7D)

* 线程池隔离:
	> 线程池隔离会为每一个依赖创建一个线程池来处理来自该依赖的请求,不同的依赖线程池相互隔离,就算依赖A出故障,导致线程池资源被耗尽,也不会影响其他依赖的线程池资源.
	* 优点: 支持排队和超时,支持异步调用
	* 缺点: 线程调度的开销, 各资源需要预先规划和分配线程池
	* 适用场景:适合耗时较长的接口场景,比如接口处理逻辑复杂,且与第三方中间件有交互,因为<font color="red">线程池模式的请求线程与实际转发线程不是同一个</font>,所以可以保证容器有足够的线程来处理新的请求
* 信号量隔离模式:
	> 初始化信号量currentCount=0,每进来一个请求需要先将currentCount自增,再判断currentCount的值是否小于系统最大信号量,小于则继续执行,大于则直接返回,拒绝请求
	* 优点:轻量,无额外的开销,只是一个简单的计数器
	* 缺点:不支持任务排队和主动超时;不支持异步调用
	* 适用场景:适合能快速响应的接口场景,不适合一些耗时较长的接口场景,因为信号量模式下的请求线程与转发处理线程是同一个,如果接口耗时过长有可能会占满容器的线程数

[![gSgx54.png](https://z3.ax1x.com/2021/04/26/gSgx54.png)](https://imgtu.com/i/gSgx54)


<br>
**<font size="4">断路器</font>** <br>

工作流程如下:
[![gSWsC4.png](https://z3.ax1x.com/2021/04/26/gSWsC4.png)](https://imgtu.com/i/gSWsC4)

1. 将远程服务调用逻辑封装进一个HystrixCommand
2. 对于每次服务调用可以使用同步或异步机制,对应执行execute()或queue()
3. 判断熔断器(circuit-breaker)是否打开或者半打开状态,如果打开跳到步骤8,进行回退策略,如果关闭进入步骤4
4. 判断线程池/队列/信号量(使用了舱壁隔离模式)是否跑满,如果跑满进入回退步骤8,否则继续后续步骤5
5. run方法中执行了实际的服务调用--服务调用发生超时时,进入步骤8
6. 判断run方法中的代码是否执行成功
	* 执行成功返回结果
	* 执行中出现错误则进入步骤8
7. 所有的运行状态(成功,失败,拒绝,超时)上报给熔断器,用于统计从而影响熔断器状态
8. 进入getFallback()回退逻辑
	* 没有实现getFallback()回退逻辑的调用将直接抛出异常
	* 回退逻辑调用成功直接返回
	* 回退逻辑调用失败抛出异常
9. 返回执行成功结果

熔断是否开启熔断器主要由依赖调用的错误比率决定的,依赖调用的错误比率=请求失败数/请求总数.Hystrix中断路器打开的默认请求错误比率为50%(这里暂时称为请求错误率),还有一个参数,用于设置在一个滚动窗口中,打开断路器的最少请求数(这里暂时称为滚动窗口最小请求数),这里举个具体的例子:如果滑动窗口最小请求数为默认20,在一个窗口内(默认10秒,统计滚动窗口的时间可以设置),收到19个请求,即使这19个请求都失败了,此时请求错误率高达95%,但是断路器也不会打开.对于被熔断的请求,并不是永久被切断,而是被暂停一段时间(默认是5000ms)之后,允许部分请求通过,若请求都是健康的(ResponseTime<250ms)则对请求健康恢复(取消熔断),如果不是健康的,则继续熔断.(这里很容易出现一种错觉:多个请求失败但是没有触发熔断.是因为在一个滚动窗口内的失败请求数没有达到打开断路器的最少请求数)<br>