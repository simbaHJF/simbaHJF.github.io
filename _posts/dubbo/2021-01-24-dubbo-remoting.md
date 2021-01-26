---
layout:     post
title:      "Dubbo Remoting 层核心接口"
date:       2021-01-24 20:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---

## 导航
[一. Dubbo Remoting 层简介](#jump1)
<br>
[二. dubbo-remoting-api 模块](#jump2)
<br>
[三. 传输层核心接口](#jump3)
<br>
[四. 总结](#jump3)



<br><br>
## <span id="jump1">一. Dubbo Remoting 层简介 </span>

[![sb9kvQ.jpg](https://s3.ax1x.com/2021/01/24/sb9kvQ.jpg)](https://imgchr.com/i/sb9kvQ)

Dubbo Remoting 层包括了 Exchange,Transport和Serialize三个子层次.Serialize层前面已经介绍过,本篇介绍 Exchange 和 Transport 两层.<br>

Dubbo Remoting 在源码中对应 dubbo-remoting模块,该模块提供了多种客户端和服务端通信的功能.Dubbo 并没有自己实现一套完整的网络库,而是使用现有的,相对成熟的第三方网络库,例如Netty,Mina或是Grizzly等NIO框架.可以根据自己的实际场景和需求修改配置,选择底层使用的NIO框架.<br>


下图展示了 dubbo-remoting 模块的结构,dubbo-remoting-api作为顶层抽象,其他每个子模块对应一个第三方 NIO 框架,具体实现顶层抽象接口，例如，dubbo-remoting-netty4 子模块使用 Netty4 实现 Dubbo 的远程通信，dubbo-remoting-grizzly 子模块使用 Grizzly 实现 Dubbo 的远程通信.<br>
[![sbTXNj.png](https://s3.ax1x.com/2021/01/24/sbTXNj.png)](https://imgchr.com/i/sbTXNj)

其中的 dubbo-remoting-zookeeper,是在基于 Zookeeper 的注册中心实现时,负责使用 Apache Curator 完成与 Zookeeper 的交互.<br>


<br><br>
## <span id="jump2">二. dubbo-remoting-api 模块 </span>

Dubbo 的 dubbo-remoting-api 是其他 dubbo-remoting-* 模块的顶层抽象,其他 dubbo-remoting 子模块都是依赖第三方 NIO 库实现 dubbo-remoting-api 模块的. 先来看一下 dubbo-remoting-api 中对整个 Remoting 层的抽象，dubbo-remoting-api 模块的结构如下图所示:
[![sbbZX6.png](https://s3.ax1x.com/2021/01/24/sbbZX6.png)](https://imgchr.com/i/sbbZX6)

来看下模块中各个包的功能:
* buffer包: 定义了缓冲区相关的接口,抽象类以及实现类.
* exchange包: 抽象了Request和Response两个概念,并为其添加很多特性.这是整个远程调用非常核心的部分.
* transport包: 对网络传输层的抽象,但它只负责抽象单向消息的传输,即请求消息由 Client 端发出, Server 端接收;响应消息由 Server 端发出, Client 端接收.有很多网络库可以实现网络传输的功能,例如 Netty、Grizzly 等, transport包是在这些网络库上层的一层抽象.
* 其它接口: Endpoint、Channel、Transporter、Dispatcher 等顶层接口放到了org.apache.dubbo.remoting 这个包下,这些接口是 Dubbo Remoting 的核心接口



<br><br>
## <span id="jump3">三. 传输层核心接口 </span>

在 Dubbo 中会抽象出一个"端点(Endpoint)"的概念,我们可以通过一个 ip 和 port 唯一确定一个端点,两个端点之间会创建TCP连接,可以双向传输数据. Dubbo 将 Endpoint 之间的 TCP 连接抽象为通道(Channel), 将发起请求的 Endpoint 抽象为客户端(Client), 将接受请求的 Endpoint 抽象为服务端(Server). 这些抽象出来的概念,也是整个 dubbo-remoting-api模块的基础,下面会逐个进行介绍.<br>

**<font size="3">Endpoint</font>** <br>
Dubbo 中 Endpoint 接口的定义如下:
[![sbv39g.png](https://s3.ax1x.com/2021/01/24/sbv39g.png)](https://imgchr.com/i/sbv39g)

**<font size="3">Channel</font>** <br>
Channel 是对两个 Endpoint 连接的抽象,好比连接两个位置的传送带,两个 Endpoint 传输的消息就好比传送带上的货物,消息发送端会往 Channel 写入消息,而接收端会从 Channel 读取消息.这与 Netty 中的 Channel 基本一致.
[![sbvXUf.png](https://s3.ax1x.com/2021/01/24/sbvXUf.png)](https://imgchr.com/i/sbvXUf)

下面是 Channel 接口的定义,可以看出两点:一个是 Channel 接口继承了 Endpoint 接口,也具备开关状态以及发送数据的能力;另一个是可以在 Channel 上附加 KV 属性
[![sbzGmn.png](https://s3.ax1x.com/2021/01/24/sbzGmn.png)](https://imgchr.com/i/sbzGmn)

**<font size="3">ChannelHandler</font>** <br>
ChannelHandler 是注册在 Channel 上的消息处理器,在 Netty 中也有类似的抽象.下图展示了 ChannelHandler 接口的定义,在 ChannelHandler 中可以处理 Channel 的连接建立以及连接断开事件,还可以处理读取到的数据、发送的数据以及捕获到的异常
[![sbzonA.png](https://s3.ax1x.com/2021/01/24/sbzonA.png)](https://imgchr.com/i/sbzonA)

**<font size="3">Codec2</font>** <br>
在 Netty 中，有一类特殊的 ChannelHandler 专门负责实现编解码功能,从而实现字节数据与有意义的消息之间的转换,或是消息之间的相互转换.在dubbo-remoting-api 中也有相似的抽象,如下所示
```
@SPI

public interface Codec2 {

    @Adaptive({Constants.CODEC_KEY})

    void encode(Channel channel, ChannelBuffer buffer, Object message) 

        throws IOException;

    @Adaptive({Constants.CODEC_KEY})

    Object decode(Channel channel, ChannelBuffer buffer)

        throws IOException;

        

    enum DecodeResult {

        NEED_MORE_INPUT, SKIP_SOME_INPUT

    }

}
```
这里需要关注的是 Codec2 接口被 @SPI 接口修饰了,表示该接口是一个扩展接口,同时其 encode() 方法和 decode() 方法都被 @Adaptive 注解修饰,也就会生成适配器类,其中会根据 URL 中的 codec 值确定具体的扩展实现类.DecodeResult 这个枚举是在处理 TCP 传输时粘包和拆包使用的,例如,当前能读取到的数据不足以构成一个消息时,就会使用 NEED_MORE_INPUT 这个枚举

**<font size="3">Client 和 RemotingServer</font>** <br>
这两个接口分别抽象了客户端和服务端.<br>

Client 和 Server 本身都是 Endpoint,只不过在语义上区分了请求和响应的职责,两者都具备发送的能力,所以都继承了 Endpoint 接口.Client 和 Server 的主要区别是 Client 只能关联一个 Channel,而 Server 可以接收多个 Client 发起的 Channel 连接.所以在 RemotingServer 接口中定义了查询 Channel 的相关方法,如下图所示:
[![sqCnBj.png](https://s3.ax1x.com/2021/01/24/sqCnBj.png)](https://imgchr.com/i/sqCnBj)


**<font size="3">Transporter</font>** <br>
Dubbo 在 Client 和 Server 之上又封装了一层Transporter 接口,其具体定义如下:
```
@SPI("netty")

public interface Transporter {

    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})

    RemotingServer bind(URL url, ChannelHandler handler) 

        throws RemotingException;

    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})

    Client connect(URL url, ChannelHandler handler) 

        throws RemotingException;

}
```
Transporter 接口上有 @SPI 注解,它是一个扩展接口,默认使用“netty”这个扩展名,@Adaptive 注解的出现表示动态生成适配器类,会先后根据"server"、"transporter"的值确定 RemotingServer 的扩展实现类,先后根据"client"、"transporter"的值确定 Client 接口的扩展实现.<br>

Transporter 接口的实现有哪些呢?如下图所示,针对每个支持的 NIO 库,都有一个 Transporter 接口实现,散落在各个 dubbo-remoting-* 实现模块中
[![sqk2SP.png](https://s3.ax1x.com/2021/01/24/sqk2SP.png)](https://imgchr.com/i/sqk2SP)

这些 Transporter 接口实现返回的 Client 和 RemotingServer 具体是什么呢?返回的是 NIO 库对应的 RemotingServer 实现和 Client 实现
[![sqAwpq.png](https://s3.ax1x.com/2021/01/24/sqAwpq.png)](https://imgchr.com/i/sqAwpq)
[![sqAsnU.png](https://s3.ax1x.com/2021/01/24/sqAsnU.png)](https://imgchr.com/i/sqAsnU)

为什么要单独抽象出 Transporter层,而不是直接让上层使用 Netty 呢?这是因为Netty、Mina、Grizzly 这些 NIO 库对外接口和使用方式不一样,如果在上层直接依赖了 Netty 或是 Grizzly,就依赖了具体的 NIO 库实现,而不是依赖一个有传输能力的抽象,后续要切换实现的话,就需要修改依赖和接入的相关代码,非常容易改出 Bug.这也不符合设计模式中的开放-封闭原则.<br>

有了 Transporter 层之后,我们可以通过 Dubbo SPI 修改使用的具体 Transporter 扩展实现,从而切换到不同的 Client 和 RemotingServer 实现,达到底层 NIO 库切换的目的,而且无须修改任何代码.即使有更先进的 NIO 库出现,我们也只需要开发相应的 dubbo-remoting-* 实现模块提供 Transporter、Client、RemotingServer 等核心接口的实现,即可接入,完全符合开放-封闭原则.<br>


**<font size="3">Transporters</font>** <br>
在最后,我们还要看一个类——Transporters,它不是一个接口,而是门面类,其中封装了 Transporter 对象的创建(通过 Dubbo SPI)以及 ChannelHandler 的处理,如下所示
```
public class Transporters {

    private Transporters() {

    // 省略bind()和connect()方法的重载

    public static RemotingServer bind(URL url, 

            ChannelHandler... handlers) throws RemotingException {

        ChannelHandler handler;

        if (handlers.length == 1) {

            handler = handlers[0];

        } else {

            handler = new ChannelHandlerDispatcher(handlers);

        }

        return getTransporter().bind(url, handler);

    }

    public static Client connect(URL url, ChannelHandler... handlers)

           throws RemotingException {

        ChannelHandler handler;

        if (handlers == null || handlers.length == 0) {

            handler = new ChannelHandlerAdapter();

        } else if (handlers.length == 1) {

            handler = handlers[0];

        } else { // ChannelHandlerDispatcher

            handler = new ChannelHandlerDispatcher(handlers);

        }

        return getTransporter().connect(url, handler);

    }

    public static Transporter getTransporter() {

        // 自动生成Transporter适配器并加载

        return ExtensionLoader.getExtensionLoader(Transporter.class)

            .getAdaptiveExtension();

    }

}

```

这里再把通过debug端点获取的Dubbo动态生成的适配器类Transporter$Adaptive的代码附上:
```
package org.apache.dubbo.remoting;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Transporter$Adaptive implements org.apache.dubbo.remoting.Transporter {
    public org.apache.dubbo.remoting.Client connect(org.apache.dubbo.common.URL arg0, org.apache.dubbo.remoting.ChannelHandler arg1) throws org.apache.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (" + url.toString() + ") use keys([client, transporter])");
        org.apache.dubbo.remoting.Transporter extension = (org.apache.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(org.apache.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.connect(arg0, arg1);
    }
    public org.apache.dubbo.remoting.RemotingServer bind(org.apache.dubbo.common.URL arg0, org.apache.dubbo.remoting.ChannelHandler arg1) throws org.apache.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (" + url.toString() + ") use keys([server, transporter])");
        org.apache.dubbo.remoting.Transporter extension = (org.apache.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(org.apache.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.bind(arg0, arg1);
    }
}
```


在创建 Client 和 RemotingServer 的时候,可以指定多个 ChannelHandler 绑定到 Channel 来处理其中传输的数据.Transporters.connect() 方法和 bind() 方法中,会将多个 ChannelHandler 封装成一个 ChannelHandlerDispatcher 对象.<br>

ChannelHandlerDispatcher 也是 ChannelHandler 接口的实现类之一,维护了一个 CopyOnWriteArraySet 集合用于存储ChannelHandler集合,它所有的 ChannelHandler 接口实现都会调用其中每个 ChannelHandler 元素的相应方法.另外,ChannelHandlerDispatcher 还提供了增删该 ChannelHandler 集合的相关方法.<br>


<br><br>
## <span id="jump4">四. 总结 </span>

到此为止,Dubbo Transport 层的核心接口就介绍完了,这里简单总结一下
* Endpoint 接口抽象了"端点"的概念，这是所有抽象接口的基础
* 上层使用方会通过 Transporters 门面类获取到 Transporter 的具体扩展实现,然后通过 Transporter 拿到相应的 Client 和 RemotingServer 实现,就可以建立(或接收)Channel 与远端进行交互了
* 无论是 Client 还是 RemotingServer，都会使用 ChannelHandler 处理 Channel 中传输的数据，其中负责编解码的 ChannelHandler 被抽象出为 Codec2 接口

整个架构如下图所示,与 Netty 的架构非常类似
[![sqEh5j.png](https://s3.ax1x.com/2021/01/24/sqEh5j.png)](https://imgchr.com/i/sqEh5j)