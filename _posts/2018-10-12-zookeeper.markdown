---
layout: post
category: "Zookeeper"
title:  "Zookeeper全攻略"
tags: [Zookeeper, 学习实战]
---

* content
{:toc}

![](http://or9cryhof.bkt.clouddn.com/images.jpeg)






## 概述

Zookeeper是apache的一款开源项目，是一款基于观察者设计模式的分布式服务管理框架，负责存储和管理大家都关心的数据，接受观察者的注册。一旦数据发生变化，Zookeeper负责通知已经注册的观察者做出相应的反应。



简单来说，**Zookeeper=文件系统+通知机制**。


### 特点

1. 一个Leader，多个Follower组成的集群。
2. 只要有半数以上的节点存货，Zookeeeper集群就能正常提供服务。
3. 全局数据一致，么个server都保存相同的数据副本，client不管连接到哪个server，数据都是一致的。
4. 数据更新具有原子性。
5. 实时性，在一定时间范围内，client总能读到最新的数据。

### 数据结构

类Unix文件系统的结构类似，整体为一棵树，每个节点为一个ZNode，每个ZNode存储1MB数据。

![](http://or9cryhof.bkt.clouddn.com/zookeeper-data-model1.png)

### 应用场景

- 统一命名服务 隐藏服务ip
- 统一配置管理
- 统一集群管理
- 服务器节点动态上线下
- 软负载均衡



## 内部原理

### 选举机制

由于半数机制的存在，所以Zookeeper适合安装奇数台服务器。

Zookeeper不能通过配置选定Master和Slaver，工作时只有一个节点为Leader，Leader和Flollower是通过内部选举机制临时产生的。

选举权重一般由**服务器ID**和**数据ID**决定。



**选举状态**：
- LOOKING，竞选状态。
- FOLLOWING，随从状态，同步leader状态，参与投票。
- OBSERVING，观察状态,同步leader状态，不参与投票。
- LEADING，领导者状态。



**5台服务器依次启动时的竞选流程**：
1. 服务器1启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于Looking。
2. 服务器2启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务器2胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING。
3. 服务器3启动，给自己投票，同时与之前启动的服务器1,2交换信息，由于服务器3的编号最大所以服务器3胜出，此时投票数正好大于半数，所以服务器3成为领导者，服务器1,2成为小弟。
4. 服务器4启动，给自己投票，同时与之前启动的服务器1,2,3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4只能成为小弟。
5. 服务器5启动，后面的逻辑同服务器4成为小弟。

![](http://or9cryhof.bkt.clouddn.com/fig01.png)



### 节点类型

- **持久(PERSISTENT)**: CS断开连接之后，创建的节点不删除
- **临时(Ephemeral)**: CS断开连接之后，创建的节点自己删除
- **持久顺序编号(PERSISTENT_SEQUENTIAL)**: 每次都会创建的节点名称后悔附加一个单调递增的计数值，由父节点维护
- **临时顺序编号(EPHEMERAL_SEQUENTIAL)**: 每次都会创建的节点名称后悔附加一个单调递增的计数值，由父节点维护，CS断开连接后自动删除



### 监听器

1. 首先要有一个main线程。
2. 在main线程中创建Zookeeper客户端，此时会创建两个线程：负责连接通信的connect线程，负责监听的listener线程。
3. 在connect线程中发送注册监听事件给Zookeeper服务器。（格式为Client:ip:port:/path）
4. Zookeeper将监听事件添加到列表中。
5. 当Zookeeper监听到有数据或者路径变化，就会发送消息给listener线程。
6. listen线程调用自己的process()方法进行处理。


常见的监听事件主要有：

- 监听数据变化: `get path watch`
- 监听子节点变化: `ls path watch`



### 写数据的流程

1. client向Zookeeper的Server1写数据，发送一个写请求。
2. 如果Server不是Leader，Server1会将收到的请求转发给Leader，Leader会将写请求广播到各个Server，各个Server写成功后通知Leader。
3. Leader收到半数以上Server写成功通知，就认为数据写成功，Leader会告诉Server1数据写成功。
4. Server1通知Client写成功，整个写操作成功。



## 安装部署

### 单机模式部署

#### 下载

```
https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/
```

将`conf/zoo_sample.cfg`修改为`conf/zoo.cfg`
并将`zoo.cfg`中的dataDir目录进行修改（原本在/tmp目录下）


各参数含义
```
#tickTime: 心跳时间，单位ms
#initLimit: LF初始通讯时限，单位：心跳帧
#syncLimit: LF同步通讯时限
#dataDir: 数据目录
```

启动Server

```
bin/zkServer.sh start
```

启动Client

```
bin/zkCli.sh
```

### 集群模式部署

有两种方式，一种为在多个服务器或hadoop集群上进行部署，第二种方式为在一台机器上部署多个zk进程，这里选择第二种。

#### 配置

在`conf`目录下分别创建三个配置文件`z1.cfg`,`z2.cfg`,`z3.cfg`

```
# zx.cfg
    tickTime=2000
    initLimit=10
    syncLimit=2
    dataDir=/usr/myenv/zookeeper-3.4.8/zx/data
    clientPort=218x
    # server.x中的“x”表示ZooKeeper Server进程的标识
    server.1=127.0.0.1:2222:2225
    server.2=127.0.0.1:3333:3335
    server.3=127.0.0.1:4444:4445
```

其中`server.A=B:C:D`为服务器配置：

- A为服务器编号
- B为服务器ip
- C为与服务器Leader进行数据同步的端口号
- D为选举Leader时执行选举通讯的端口号

其余配置与单机部署含义相同，不过需要注意在同一机器上部署多个进程时不能共用一个`dataDir`和`clientPort`。

在相应的数据目录下分别创建`myid`文件并储存相应的服务器编号。

#### 启动服务器
```sh
bin/zkServer.sh start conf/z1.cfg
bin/zkServer.sh start conf/z2.cfg
bin/zkServer.sh start conf/z3.cfg
```

#### 查看服务器状态
```sh
➜  zookeeper-3.4.10 bin/zkServer.sh status conf/z1.cfg
ZooKeeper JMX enabled by default
Using config: conf/z1.cfg
Mode: follower
➜  zookeeper-3.4.10 bin/zkServer.sh status conf/z2.cfg
ZooKeeper JMX enabled by default
Using config: conf/z2.cfg
Mode: leader
➜  zookeeper-3.4.10 bin/zkServer.sh status conf/z3.cfg
ZooKeeper JMX enabled by default
Using config: conf/z3.cfg
Mode: follower
```

#### 客户端连接
```
bin/zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

## demo - 动态获取集群服务器host列表

Zookeeper有一套简单灵活的客户端api，可以借助这些api注册监听Zookeeper事件并获取响应节点数据以实现不同功能。

目前常用的场景是动态获取集群中的服务器host以实现软负载均衡。

### 添加pom依赖
```java
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.10</version>
</dependency>
```

### 服务器注册

每个服务器启动时需要连接Zookeeper并在`/servers`节点下新增临时顺序编号节点，并写入自己的host，当服务器下线时该节点自动删除。

```java
import org.apache.zookeeper.*;

import java.io.IOException;

public class TargetServer {

    private ZooKeeper zkClient;

    private void connect() throws IOException {
        String connectString = "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183";
        int sessionTimeout = 2000;

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {

            }
        });
    }

    private void register(String hostname) throws KeeperException, InterruptedException {

        String path = zkClient.create("/servers/server", hostname.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

        System.out.println(hostname + " register on zookeeper success!");
    }

    private void run() throws InterruptedException {

        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {

        TargetServer server = new TargetServer();
        server.connect();
        server.register(args[0]);
        server.run();
    }
}

```

### 客户端监听

客户端启动时需连接Zookeeper并获取`/servers`节点下的可用节点，拿到host列表并注册监听，当host有变动时更新自己的host列表。

```java
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class ConnectClient {

    private ZooKeeper zkClient;

    private void connect() throws IOException {
        String connectString = "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183";
        int sessionTimeout = 2000;

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                try {
                    getChildren();
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    private void getChildren() throws KeeperException, InterruptedException {

        List<String> children = zkClient.getChildren("/servers", true);

        ArrayList<String> hosts = new ArrayList<String>();

        for (String child : children) {
            byte[] data = zkClient.getData("/servers/" + child, false, null);
            hosts.add(new String(data));
        }

        System.out.println(hosts);
    }

    private void run() throws InterruptedException {
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        ConnectClient client = new ConnectClient();
        client.connect();
        client.getChildren();
        client.run();
    }
}
```

(End)
