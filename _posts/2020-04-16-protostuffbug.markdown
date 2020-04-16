---
layout: post
category: "问题排查"
title:  "一个 Protostuff 序列化问题"
tags: [protostuff, 序列化]
---

* content
{:toc}

![596-800x300.jpg](https://i.loli.net/2019/09/27/edshNbA8Vn3aKOq.jpg)

记录一个由于 Protostuff 序列化缺陷导致 RPC 调用失败的问题。





## 问题描述
部门内使用了自研 RPC 框架（类似 Dubbo），Sever 端定义方法简化如下
```java
interface UserService {
    void resolveUser(Long userId, Long optId, Long userType);
}
```
客户端在进行调用时，`opdId`参数传入了`null`（这里客户端没有进行空值校验也是有问题的，先不谈）
```java
userService.resolveUser(1L, null, 1L);
```
服务端接收到请求后，参数位置顺序错误，`null`被赋给了`userType`
```java
userId: 1L
optId: 1L
userType: null
```

## 排查思路
在确认客户端、服务端接口版本一致后，怀疑是 RPC 框架内部逻辑问题。
框架内 RPC 请求被封装为`RpcRequest`类，保存`requestId`、`methodName`、`paramters`等信息。
```java
public final class RpcRequest {
    private String id;
    private String methodName;
    private Object[] paramters;

    //...
}
```
分别在客户端、服务端对`RpcRequest`进行序列化和反序列化的部分进行断点调试，发现`RpcRequest`对象在序列化之前`paramters`数组保存了正确的参数信息`[1L, null, 1L]`，在服务端反序列化之后，`paramters`数组的顺序发生了变化`[1L, 1L, null]`，到这里可以确认问题出在序列化上。

## 继续深入
RPC 框架使用的序列化工具是`Protostuff`，由`Protobuf`发展而来，保留其高性能的同时提供了更简单的使用方法。
在本地模拟框架内`Protostuff`的序列化方式如下：
```java
public class ProtoStuffUtil {
    //避免每次序列化都重新申请Buffer空间
    private static LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
    //缓存Schema
    private static Map<Class<?>, Schema<?>> schemaCache = new ConcurrentHashMap<Class<?>, Schema<?>>();
    //序列化方法，把指定对象序列化成字节数组
    @SuppressWarnings("unchecked")
    public static <T> byte[] serialize(T obj) {
        Class<T> clazz = (Class<T>) obj.getClass();
        Schema<T> schema = getSchema(clazz);
        byte[] data;
        try {
            data = ProtostuffIOUtil.toByteArray(obj, schema, buffer);
        } finally {
            buffer.clear();
        }
        return data;
    }
    //反序列化方法，将字节数组反序列化成指定Class类型
    public static <T> T deserialize(byte[] data, Class<T> clazz) {
        Schema<T> schema = getSchema(clazz);
        T obj = schema.newMessage();
        ProtostuffIOUtil.mergeFrom(data, obj, schema);
        return obj;
    }
    @SuppressWarnings("unchecked")
    private static <T> Schema<T> getSchema(Class<T> clazz) {
        Schema<T> schema = (Schema<T>) schemaCache.get(clazz);
        if (schema == null) {
            schema = RuntimeSchema.getSchema(clazz, new DefaultIdStrategy(IdStrategy.DEFAULT_FLAGS |
                    IdStrategy.ALLOW_NULL_ARRAY_ELEMENT));
            if (schema != null) {
                schemaCache.put(clazz, schema);
            }
        }
        return schema;
    }
}
```
之后构建测试 Demo：
```java
@Data
@ToString
public class Request {

    private Object[] params;

    public static void main(String[] args) {
        Request request = new Request();
        request.setParams(new Object[]{1, null, 2});

        byte[] bytes = ProtoStuffUtil.serialize(request);
        Request requestDecode = ProtoStuffUtil.deserialize(bytes, Request.class);

        System.out.println(requestDecode);
    }
}
```
测试结果：
```java
Request(params=[1, 2, null])
```
可见，**Protostuff 对包含空值的 Object[] 数组的反序列会存在顺序问题**，这也是我们 Rpc 调用参数顺序倒置的原因。

## 解决方案
经过一些调研发现，github 上也存在其他 Rpc 框架有类似问题（https://github.com/fengjiachun/Jupiter/issues/73），并且 protostuff作者也不认为这是一个bug, 不会修复。

同样，issue 也给出了 workaround 的方案，即采用空对象标记`null`元素。

但作为框架的使用方，我只能将参数封装为 DTO 对象绕过这一问题，除此之外，不但作为 Server 要进行入参校验，作为 Client 同样要对请求的参数负责，才能尽量避免故障。

（End）
