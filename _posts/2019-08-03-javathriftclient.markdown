---
layout: post
category: "Java"
title:  "优雅地在 Java 中进行 Thrift 调用"
tags: [Java, Thrift]
---
* content
{:toc}

![907-800x300.jpg](https://i.loli.net/2019/08/03/j2eB1Wf5KZ4piHn.jpg)

项目需要引入 Thrift Client 进行第三方调用，引入的过程本身是很简单的，但写的时候还是尽量抽象一些公共逻辑，避免过多的重复代码，这里记录一种比较优雅的方式。





## QuickStart

我们先以一个简单的 thrift 接口为例进行最简单的接入，thrift 接口如下：
```
service SimpleService {
    void ping();
}
```
通过 thrift compiler 生成 java 代码
```shell
thrift --gen java simpleservic.thrift
```
生成的 java 代码中与客户端相关的包括下面几部分（省略了具体内容）：
```java
public class SimpleService {
　　public interface Iface {}
　　public interface AsyncIface {}
 
　　public static class Client {}
　　public static class AsyncClient {}   
}
```
两个接口分别是接口方法的同步和异步调用接口，实现类是具体 Client 实现，其已经生成了具体的读写逻辑，只需要在实例化时传入具体的 protocol 即可完成接口调用，下面我们在项目中引入 client。

首先引入依赖，注意版本与 thrift compiler 保持一致。
```xml
<dependency>
    <groupId>org.apache.thrift</groupId>
    <artifactId>libthrift</artifactId>
    <version>0.12.0</version>
</dependency>
```

下面引入 thrift client 进行调用，创建 transport 和 protocol，实例化 client 即可。
```java
TFastFramedTransport transport = new TFastFramedTransport(new TSocket(host, port));
TBinaryProtocol protocol = new TBinaryProtocol(transport);
SimpleSerice.Client client = new SimpleService.Client(protocol);
try {
    transport.open();
    client.ping();
} finally {
    transport.close();
}
```
这里解释下各组件的含义：

### Transport

通信层协议，使用`TTransport`接口来描述，提供了用于数据传输的方法：
- open
- close
- read
- write
- flush

Thrift 提供了几种 transport 类供使用：
- **TSocket**：BIO
- **TFramedTransport**：NIO
- **TMemoryTransport**：内存I/O
- **TZlibTransport**：Zlib 压缩传输，Java 中没有实现

需要注意的是，transport 客户端要和服务端对应，比如如果要对 NIO 模型的服务端进行调研，transport 就必须使用 `TFramedTransport`或`TFastFramedTransport`。

### Protocol

传输层协议，引用 transport 并提供 thrift 的序列化协议，Thrift 也提供了几种常用的传输协议：
- **TBinaryProtocol**：二进制协议
- **TCompactProtocol**：压缩格式
- **TJSONProtocol**：JSON 协议

同样的，客户端要和服务端的传输层协议对应。

## 使用 JDK 动态代理

当然，上面的写法只是简单实现了功能，在实际开发中，要对上面的客户端实例化的逻辑进行抽象，隐藏 transport、protocol 这些细节，并做好 transport 生命周期的管理。对于业务代码来讲，只需要引用`Iface`接口，而实际则需要创建并使用`Client`发起调用，JDK 动态代理正好可以满足这个需求。

首先创建代理类，这里只需要引用 thrift 接口中的 Service 类，并通过引用发现内部 Client 类。
```java
public class ThriftClientProxy implements InvocationHandler {

    private Class clazz;
    private Class clientClazz;
    private String host;
    private int port;

    public Object newInstance(Class clazz, String host, int port) {
        this.clazz = clazz;
        this.host = host;
        this.port = port;
        this.clientClazz = Class.forName(classs.getName() + "$Client");

        return Proxy.newProxyInstance(
            getClass().getClassLoader(),
            clientClazz.getInterfaces(),
            this
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TFastFramedTransport transport = new TFastFramedTransport(new TSocket(host, port));
        TBinaryProtocol protocol = new TBinaryProtocol(transport);
        Class[] classes = {TProtocol.class};
        try {
            transport.open();
            return method.invoke(clientClazz.getConstructor(classes).newInstance(protocol), args);
        }
        finally {
            transport.close();
        }
    }
}
```

之后将相应的 Iface 注册到 IOC 容器中即可。
```java
@Configuration
public class ThriftClientConfiguration {

    @Value("${simple.service.host}")
    private String host;

    @Value("${simple.service.port}")
    private int port;

    @Bean
    public SimpleService.Iface thriftClient() {
        return (SimpleService.Iface) new ThriftClientProxy().newInstance(host, port);
    }
}
```
业务代码中只需要注入 SimpleService.Iface，并直接调用接口方法，而完全不需要关注内部逻辑实现。

当然，这里只是介绍了比较简单的场景，实际场景中可能存在的连接池复用、负载均衡、调用超时等需求，都可以在动态代理中增加相应的逻辑，这里就不再赘述了。

（ End ）

> 参考资料
> - [Thrift 简易入门与实战](https://segmentfault.com/a/1190000008606491)
> - [Thrift 服务化 Client的改造](https://www.cnblogs.com/mumuxinfei/p/3876187.html)