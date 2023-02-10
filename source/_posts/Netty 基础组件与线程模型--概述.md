---
title: Netty 基础组件与线程模型--概述
categories:
- Netty
date: 2019-9-3
updated: 2019-9-3
---

# Netty 基础组件与线程模型--概述

即使仅仅谈论这两个话题：Netty 组件和 Netty 线程模型，也不是一两篇文章可以 cover 的。这里做一个小尝试，也对这几天看的 *Netty in Action* 总结一下。

## Netty 想要解决什么问题

一项新技术/开源项目，问问自己，这个技术想要解决的是什么问题，是如何解决的。

以我目前的水平回答，是为了弥补 Java 标准库在 I/O 上的不足。这种不足我现在看来有两部分：性能与易用性。

性能方面，作为标准库，API 稳定、统一的需求更重要，从而无法在提升性能上下狠功夫。Netty 则在各种角落实行了优化。最明显的是对垃圾回收与内存分配的修改，使得 ByteBuf 一定程度由使用者手动管理；以及针对不同平台提供与底层实现紧密相关的优化，比如针对 Linux 的 EpollEventLoopGroup，从而能提供边缘触发，而非 Java NIO 默认的水平触发。另外，据 Norman Maurer [自己所说](https://stackoverflow.com/a/36058105/8144090)，还有更低的 GC。其它细节包括使用 OpenSSL 替换 Java 标准库，等等。

另一方面，NIO 在使用上不够直观。仅仅 `flip` 这个方法的设计就很迷惑——像 Netty 一样为读和写单独设置两个指针多方便。

## Netty 自测问题

看完 Netty in Action 之后我反问自己都了解到了些什么，想到了如下的问题：

`EventLoop`  与 `EventLoopGroup` 的区别，之间的关系是什么？EventLoopGroup 拥有几个线程？EventLoop 呢？

EventLoop 与 Channel 是什么关系？

ChannelHandler 与 ChannelPipeline 是什么关系？

ChannelHandlerAdapter 对 ChannelHandler 区别是什么，有何改进？

Channel、ChannelHandler 生命周期？

我发现很多问题自己居然都给不出回答，然后立刻回去看书的三、六、七章，有一种开窍的感觉。之前看第一遍时无感，并且不知道重点的内容，这一次都变成了问题的答案，看的过程也津津有味起来。

## 简单梳理

简单梳理一下基础的概念，下图来自《Netty in Action》，侵删：

![Channel EventLoop and EventLoopGroup](/content/images/2019/Channel EventLoop and EventLoopGroup.png)



- Channel 就是一种“传输（transport）”，一个 Socket 就属于 Channel，所以最简单可以理解为，Channel 就是一个 Socket；
- 一个 Channel 在它的生命周期内只注册于一个 EventLoop；
- 一个 EventLoop 在它的生命周期中只与一个 Thread 绑定。实际上，最常用的实现 NioEventLoop 扩展自 SingleThreadEventLoop；
- 一个 EventLoop 可能会被分配给一个或多个 Channel（这样一来，不必每个 Socket 连接就分配一个线程，用有限的线程支撑更多的连接）；

这种设计消除了同步（synchronization）的需求，因为一个 Channel 的所有 I/O 操作都是由同一个线程（异步地）完成的。

- ChannelPipeline 提供了 ChannelHandler 链的容器

## 线程模型

### Reactor pattern & Proactor pattern

在学习阅读 Netty 的时候，我意识到书的作者以及文档作者都假定了，读者对于所谓的 Proactor 模式、Reactor 模式有所了解。所以我认为有必要先简要了解一下这两种模式。

[Reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern) 有**一个**主线程，从多个 input 读取服务请求，然后把这些请求**同步地**分发给对应的服务处理线程。通常主线程使用同步的 `select` 来监听多个文件描述符，当一个阻塞操作，比如 `read()` 准备完毕，`select` 获得通知，就将其转发给服务处理线程。

关于 [Proactor pattern](<https://en.wikipedia.org/wiki/Proactor_pattern>)：

> The proactor pattern can be considered to be an [asynchronous](https://en.wiktionary.org/wiki/asynchronous) variant of the [synchronous](https://en.wikipedia.org/wiki/Synchronization_(computer_science)) [reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern).
>
> -- Wikipedia

Proactor 模式可以视作 Reactor 模式的异步形式。耗时长的任务异步地完成，在结束时发送通知。

### Netty 的 Boss-Worker EventLoopGroup

了解了有这种主-从线程池的区分之后，就能理解，为什么很多 Netty 示例程序中 Server 的 `ServerBootStrap#group(EventLoopGroup, EventLoopGroup)` 会有绑定两个 EventLoopGroup。即使某些简单的 demo 实现中只绑定了一个，如： `group(EventLoopGroup)`，这一方法其实只是绑定主从 EventLoopGroup 的重载方法，源码如下：

```java
@Override
public ServerBootstrap group(EventLoopGroup group) {
	return group(group, group);
}
```

官方 [User Guide](<https://netty.io/wiki/user-guide-for-4.x.html>) 的解释如下：

> The first one, often called 'boss', accepts an incoming connection. The second one, often called 'worker', handles the traffic of the accepted connection once the boss accepts the connection and registers the accepted connection to the worker. 

第一个是被称为 boss 的 EventLoopGroup，负责接受连接，第二个则被称为 worker，负责处理由 boss 分配给它的已接受的连接。