# simple-rpc-framework


## 前言

目前只实现了 RPC 框架最基本的功能

## 介绍

是一款基于 Netty+Kyro+Zookeeper 实现的 RPC 框架。


### 该RPC项目的结构

该框架使用示意图如下图所示：

![](./images/rpc-architure.png)

服务提供端 Server 向注册中心注册服务，服务消费者 Client 通过注册中心拿到服务相关信息，然后再通过网络请求服务提供端 Server。


**一般情况下， RPC 框架不仅要提供服务发现功能，还要提供负载均衡、容错等功能，这样的 RPC 框架才算真正合格的。**


![](./images/rpc-architure-detail.png)

1. **注册中心** ：注册中心首先是要有的，使用 Zookeeper。注册中心负责服务地址的注册与查找，相当于目录服务。服务端启动的时候将服务名称及其对应的地址(ip+port)注册到注册中心，服务消费端根据服务名称找到对应的服务地址。有了服务地址之后，服务消费端就可以通过网络请求服务端了。
2. **网络传输** ：既然要调用远程的方法就要发请求，请求中包含你调用的类名、方法名以及相关参数！基于 NIO 的 Netty 框架。
3. **序列化** ：因为涉及到网络传输就一定涉及到序列化，因JDK 自带的序列化效率低并且有安全漏洞，这里使用的是kyro
4. **动态代理** ： 另外，动态代理也是需要的。因为 RPC 的主要目的就是让我们调用远程方法像调用本地方法一样简单，使用动态代理可以屏蔽远程方法调用的细节比如网络传输。也就是说当你调用远程方法的时候，实际会通过代理对象来传输网络请求，不然的话，怎么可能直接就调用到远程方法呢？
5. **负载均衡** ：负载均衡也是需要的。为啥？举个例子我们的系统中的某个服务的访问量特别大，我们将这个服务部署在了多台服务器上，当客户端发起请求的时候，多台服务器都可以处理这个请求。那么，如何正确选择处理该请求的服务器就很关键。假如，你就要一台服务器来处理该服务的请求，那该服务部署在多台服务器的意义就不复存在了。负载均衡就是为了避免单个服务器响应同一请求，容易造成服务器宕机、崩溃等问题，我们从负载均衡的这四个字就能明显感受到它的意义。


### 项目基本情况

- **使用 Netty实现网络传输；**
- **使用开源的序列化机制 Kyro替代 JDK 自带的序列化机制；**
- **使用 Zookeeper 管理相关服务地址信息**
- Netty 重用 Channel 避免重复连接服务端
- 使用 `CompletableFuture` 包装接受客户端返回结果（之前的实现是通过 `AttributeMap` 绑定到 Channel 上实现的）
- **使用Netty 心跳机制** : 保证客户端和服务端的连接不被断掉，避免重连。
- **客户端调用远程服务的时候进行负载均衡** ：调用服务的时候，从很多服务地址中根据相应的负载均衡算法选取一个服务地址。ps：目前实现了随机负载均衡算法与一致性哈希算法。
- **处理一个接口有多个类实现的情况** ：对服务分组，发布服务的时候增加一个 group 参数即可。
- **集成 Spring 通过注解注册服务和服务消费**
- **增加服务版本号** ：建议使用两位数字版本，如：1.0，通常在接口不兼容时版本号才需要升级。为什么要增加服务版本号？为后续不兼容升级提供可能，比如服务接口增加方法，或服务模型增加字段，可向后兼容，删除方法或删除字段，将不兼容，枚举类型新增字段也不兼容，需通过变更版本号升级。
- **客户端与服务端通信协议（数据包结构）重新设计** ，可以将原有的 `RpcRequest`和 `RpcReuqest` 对象作为消息体，然后增加如下字段
  - **魔数** ： 通常是 4 个字节。这个魔数主要是为了筛选来到服务端的数据包，有了这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接以节省资源。
  - **序列化器编号** ：标识序列化的方式，比如是使用 Java 自带的序列化，还是 json，kyro 等序列化方式。
  - **消息体长度** ： 运行时计算出来。
- **设置 gzip 压缩**


