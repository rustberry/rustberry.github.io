---
title: ZooKeeper：分布式过程协同技术详解 -- 读书笔记
categories:
- ZooKeeper
---

# ZooKeeper：分布式过程协同技术详解 -- 读书笔记

> 要点：第九章 Zab 协议、ZK 对写事务日志的优化。

## Installation & configuration

Todo: `/tmp/zookeeper` --> `/var/lib/zookeeper`

## Summary

#### chapter 2

types of `znode`:

- persistent, ephemeral, persistent_sequential, ephemeral_sequential；
- 持久节点、临时节点、持久有序节点、临时有序节点；
- 被创建为有序的节点被分配一个递增的整数，总而，形成了一个全局唯一的名字；
- 节点除了拥有名字（父节点名字 + 自身名字），还可以像文件夹一样，存有数据；

通知

Zookeeper 中可以在节点上注册事件，而不需要轮询。事件可以包括 znode 数据的变化、子节点的变化、znode 的创建删除。

会话 session

客户端通过 TCP 与服务端通信。当此前会话无法继续（掉线，超过延迟时限），客户端的会话会被透明地转移到下一个服务器。

Zookeeper 保证，在同一个会话中的请求是以 FIFO 的顺序执行。但是如果客户端拥有多个并发的会话，则不保证。

会话状态

NOT_CONNECTED --> CONNECTING <==> CONNECTED --> CLOSED

客户端不能声明自己的会话到期（timedout），但是可以显示关闭会话。

进阶：事务标识符 zkid 在客户端连接中的使用。

##### 端口

Zookeeper 可以配置三种 TCP 端口。前两种是关于服务端的，第三种在 `zoo.cfg` 中称为  `clientPort`。一个仲裁模式（quorum）的配置示例如下：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=./data
clientPort=2181

server.1=127.0.0.1:2222:2223
server.2=127.0.0.1:3333:3334
server.3=127.0.0.1:4444:4445
```

在对服务器的配置 `server.1=127.0.0.1:2222:2223` 中，第一个端口 `2222` 是*仲裁通信*，follower 通过这个端口来接受 leader 传达的更新指令；第二个端口用于 leader 选举。

#### connectString

```shell
zookeeper/bin/zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

上面的 `127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183` 就是所谓连接字符串（connectString）。不仅在命令行的 `zkCli` 中使用，API 库也是同样的道理。连接字符串列出了本客户端可以连接的所有服务器。其中的端口，就是在各个 Zookeeper 服务器的 `zoo.cfg` 配置文件中的 `clientPort`。



#### chap 3 使用 Zookeeper API 库

创建 Zookeeper 句柄（handle）；实现 Watcher、认识 Zookeeper 自动管理与服务端的断连；同步地创建节点 `zk.create(...)`；同步地获取 znode 中的数据 `byte[] getData(...)`、处理 `ConnectionLossException` ；异步创建节点、实践：异步创建元数据 path；`StatCallback`；

关于回调的注意：回调函数中的 `Object ctx` 在需要在回调中再次调用自身的场景中被使用到，如处理返回码为 `ConnectionLossException` 时，再次调用自身这个回调函数。

chap 4 Dealing with State Change

当一个监视点（watch）被一个事件（event）触发，就会产生一个通知（notification）。



在某些 ZooKeeper 的封装库（wrapper ）中，对 ConnectionLoss 的处理就是自动重新发送相应指令。某些情况这会带来大问题，比如通过 `create` 创建 `/master` 节点来获取领导权。如果客户端在 `create` 成功之后丢失连接，此时 wrapper 库自动重发指令，就会收到创建失败的响应，因为节点之前已经被创建好了。但是如果不知道这一细节，应用就会认为是其它节点已经创建了 master 节点获得了领导权。

#### chap 9 原理

ZK 要求，新的 Quorum 数量的服务器中，必须有一台以上的机器是属于上一个群首的 Quorum。

ZK 服务器的分类方式：分为 Leader 和 Follower，或者分为 Participant 和 Observer。Observer 和 Follower 可以统称为 Learner，因为它们“学习”群首的指示。

对于 Follower 有两类广播：

- `proposal`：包含有 transaction 的内容。
- `commit`：只包含 zxid。

对于 Observer 只有一类：

- `INFORM`：已被 commit 了的 proposal 的内容。

##### ZAB 协议

- 两阶段提交：对于一个写请求操作，群首广播出一个 Proposal。对于这个 Proposal，追随者收到之后返回 ACK 消息，通知群首其已接收此 Proposal。在收到达到仲裁数量（群首自己也算在内）的服务器发送的 ACK 之后，群首发出 `commit` 的广播。
- Zab 提供的顺序保障：
  - 如果群首按顺序广播了事务 `T1` 和 `T2`，那么每个服务器在 commit `T2` 之前保证 `T1` 已经被提交；
  - 如果某个服务器按照 `T1` 、`T2` 的顺序提交事务，那么其它服务器也保证是按这个顺序提交的。
- epoch 时间戳：每次群首选举中递增 `1`
- Zab 提供的脑裂保障：
  - 新群首必须要确保所有之前的时间戳内需要提交的事务被全部提交完，然后才开始新的广播；
  - 在任何时间点，都不会同时出现两个被仲裁数量支持的群首。

由于在选举群首时，选举上的是 `zxid` 最大的服务器，我们就不需要将最新状态从跟随服务器更新到群首服务器，因为群首就是代表最新状态；我们所需要的，就是直接发送更新到追随者。

##### ZK 有效写入事务日志

ZK 如何有效写入事务日志：写事务日志是写请求操作的关键，ZK 用了两种方式来优化：1）组团提交 2）预先填充（padding）。

实际上，对于组团提交，就是在需要 flush 到磁盘（来保证事务已被持久化）时，把所有队列中的事务都*顺带* flush 到磁盘。注意，ZK 服务器只有在事务已经写入日志之后才会确认对应的 Proposal，也就是在确认事务之前，该事务已经被持久化到磁盘了。为了保证在 `commit` 方法返回后写入的数据已经存在于磁盘（而不是操作系统的写缓冲区），磁盘写缓存必须被关闭。

**预先填充（padding）**是指预先为日志文件分配多余的磁盘块。在 Linux 系统中，一个文件中的数据储存在一个个分配给它的块（block）中。当向文件中写入时，如果当前分配的磁盘块用尽了，就要花时间请求分配新的磁盘块。这种请求需要修改该文件所对应的元数据，因此预先分配就会减少这一次磁盘寻道。

##### 会话

Standalone 模式下的单独服务器与仲裁模式（Quorum）下的群首服务器负责跟踪*所有会话*。追随者只是将客户端的会话信息转发给群首。

##### 监视点

服务端不会持久化监视点，因此监视点的数据在*客户端*保存。在掉线重连之后，客户端将监视点的信息同步到服务端

### 机翻问题。。

机翻

P58：

异步方法调用会简单化队列对 Zookeeper 服务器的请求，并在另一个线程中传输请求。

当接收到响应信息，这些请求就会在一个专用回调线程中被处理。

为了保持顺序，只会有一个单独的线程按照接收顺序处理响应包

原文：

The asynchronous method simply queues the request to the ZooKeeper server. Transmission happens on another thread. When responses are received, they are processed on a dedicated callback thread. To preserve order, there is a single callback thread and responses are processed in the order they are received.

我认为好的翻译：

这个异步方法只是简单地把请求加入 ZooKeeper 服务器的请求队列中。传输请求的任务由另一个线程完成。

收到响应之后，**响应**由一个专门的回调线程处理。

为了保证顺序，只会有一个单独的回调线程，按照接收到的顺序处理响应。



Returns the result of the call：返回调用的结构。





### 关于 `InterruptedException`

使用 ZK 的 API 库时会遇到不少的方法，抛出 `InterruptedException`。作者在书中也表示很无奈，没有一劳永逸的解决办法，还是要取决于具体状况，来决定是让这个异常沿着调用栈上浮，还是其它办法：

> In our example, we simply pass the InterruptedException to the caller and thus let it bubble up. Unfortunately, in Java there aren’t clear guidelines for how to deal with thread interruption, or even what it means. Sometimes the interruptions are used to signal threads that things are being shut down and they need to clean up. In other cases, an interruption is used to get control of a thread, but execution of the application continues.
> Our handling of InterruptedException depends on our context. If the InterruptedException will bubble up and eventually close our zk handle, we can let it go up the stack and everything will get cleaned up when the handle is closed. If the zk handle is not closed, we need to figure out if we are the master before rethrowing the exception or asynchronously continuing the operation. This latter case is particularly tricky and requires careful design to handle properly.

