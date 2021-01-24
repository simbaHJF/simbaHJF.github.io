---
layout:     post
title:      "Dubbo Serialize层"
date:       2021-01-23 23:30:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - dubbo

---

## 导航
[一. Dubbo Serialize层](#jump1)





<br><br>
## <span id="jump1">一. Dubbo Serialize层</span>

[![sb9kvQ.jpg](https://s3.ax1x.com/2021/01/24/sb9kvQ.jpg)](https://imgchr.com/i/sb9kvQ)

在 Dubbo 的整体架构设计图中,我们可以看到 Remoting 层中包括了 Exchange、Transport和Serialize 三个子层次.本篇介绍的Serialize层,位于最底层.<br>

Dubbo 为了支持多种序列化算法,单独抽象了一层 Serialize 层,在整个 Dubbo 架构中处于最底层,对应的模块是 dubbo-serializetion模块.<br>

dubbo-serialization 模块的结构如下图所示:
[![sHir6g.png](https://s3.ax1x.com/2021/01/23/sHir6g.png)](https://imgchr.com/i/sHir6g)

dubbo-serialization-api 模块中定义了 Dubbo 序列化层的核心接口,其中最核心的是 Serialization 这个接口,它是一个扩展接口,被 @SPI 接口修饰,默认扩展实现是 Hessian2Serialization.Serialization 接口的具体如下:

```
@SPI("hessian2")	// 被@SPI注解修饰，默认是使用hessian2序列化算法
public interface Serialization {
    
     // 获取ContentType的ID值，是一个byte类型的值，唯一确定一个算法
    byte getContentTypeId();
    
     // 每一种序列化算法都对应一个ContentType，该方法用于获取ContentType
    String getContentType();
    
     // 创建一个ObjectOutput对象，ObjectOutput负责实现序列化的功能，即将Java对象转化为字节序列
    @Adaptive
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    
     // 创建一个ObjectInput对象，ObjectInput负责实现反序列化的功能，即将字节序列转换成Java对象
    @Adaptive
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```


Dubbo 提供了多个 Serialization 接口实现,用于接入各种各样的序列化算法,如下图所示:
[![sHOMdS.png](https://s3.ax1x.com/2021/01/24/sHOMdS.png)](https://imgchr.com/i/sHOMdS)

这里我们以默认的 hessian2 序列化方式为例,介绍 Serialization 接口的实现以及其他相关实现. Hessian2Serialization 实现如下所示:
```
public class Hessian2Serialization implements Serialization {

    public byte getContentTypeId() {

        return HESSIAN2_SERIALIZATION_ID; // hessian2的ContentType ID

    }

    public String getContentType() { // hessian2的ContentType

        return "x-application/hessian2";

    }

    public ObjectOutput serialize(URL url, OutputStream out) throws IOException { // 创建ObjectOutput对象

        return new Hessian2ObjectOutput(out);

    }

    public ObjectInput deserialize(URL url, InputStream is) throws IOException { // 创建ObjectInput对象

        return new Hessian2ObjectInput(is);

    }

}

```

Hessian2Serialization 中的 serialize() 方法创建的 ObjectOutput 接口实现为 Hessian2ObjectOutput,继承关系如下图所示:
[![sHOszR.png](https://s3.ax1x.com/2021/01/24/sHOszR.png)](https://imgchr.com/i/sHOszR)

在 DataOutput 接口中定义了序列化 Java 中各种数据类型的相应方法,如下图所示,其中有序列化 boolean、short、int、long 等基础类型的方法,也有序列化 String、byte[] 的方法
[![sHO7QI.png](https://s3.ax1x.com/2021/01/24/sHO7QI.png)](https://imgchr.com/i/sHO7QI)

ObjectOutput 接口继承了 DataOutput 接口,并在其基础之上,添加了序列化对象的功能,具体定义如下图所示,其中的 writeThrowable()、writeEvent() 和 writeAttachments() 方法都是调用 writeObject() 方法实现的
[![sHXZY4.png](https://s3.ax1x.com/2021/01/24/sHXZY4.png)](https://imgchr.com/i/sHXZY4)

Hessian2ObjectOutput 中会封装一个 Hessian2Output 对象,需要注意,这个对象是 ThreadLocal 的,与线程绑定.在 DataOutput 接口以及 ObjectOutput 接口中,序列化各类型数据的方法都会委托给 Hessian2Output 对象的相应方法完成,实现如下
```
public class Hessian2ObjectOutput implements ObjectOutput {

    private static ThreadLocal<Hessian2Output> OUTPUT_TL = ThreadLocal.withInitial(() -> {

        // 初始化Hessian2Output对象

        Hessian2Output h2o = new Hessian2Output(null);        h2o.setSerializerFactory(Hessian2SerializerFactory.SERIALIZER_FACTORY);

        h2o.setCloseStreamOnClose(true);

        return h2o;

    });

    private final Hessian2Output mH2o;

    public Hessian2ObjectOutput(OutputStream os) {

        mH2o = OUTPUT_TL.get(); // 触发OUTPUT_TL的初始化

        mH2o.init(os);

    }

    public void writeObject(Object obj) throws IOException {

        mH2o.writeObject(obj);

    }

    ... // 省略序列化其他类型数据的方法

}

```

Hessian2ObjectInput 具体的实现与 Hessian2ObjectOutput 类似:在 DataInput 接口中实现了反序列化各种类型的方法,在 ObjectInput 接口中提供了反序列化 Java 对象的功能,在 Hessian2ObjectInput 中会将所有反序列化的实现委托为 Hessian2Input.