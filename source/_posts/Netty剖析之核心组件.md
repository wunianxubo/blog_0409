---
title: Netty剖析之核心组件
date: 2018-06-13 21:53:00  
tags: [Netty]    
categories: Netty
---

## 前言
在之前的博文《Java NIO原理解析》里面，主要讲了NIO的一些原理，几种IO模型的对比，NIO中三个重要的组件Channel，Selector以及Buffer，通过Selector单线程轮询状态实现了IO复用机制，这些在这就不重新讨论了。  

然而，原生的NIO方法比较复杂，Netty进行了很好的封装，提供更易于使用的API，性能更好，健壮性、安全性也很好，在很多地方都能见到Netty的身影，比如阿里的Dubbo分布式框架，底层就使用Netty作为网络通信框架，接下来会用几篇博文来详细讲下Netty这个高性能的网络框架。  
<!-- more -->
## 定义
Netty是什么呢？Netty是一款**异步**的**事件驱动**的网络应用程序框架，支持快速地开发可维护的**高性能**面向协议的服务器和客户端。  

从下面Netty的框架图可以看出Netty支持很多协议，安全协议的支持，高性能Google序列化协议protobuf的支持等等，可以说很强大了。  

![image](http://osrmzp0jr.bkt.clouddn.com/netty.png)  

**异步，事件驱动，高性能**是Netty关键词。  

- **异步**就是发出请求之后就会返回，而不用等到请求都完成后再返回，异步主要通过回调通知来实现，即事件完成后会有回调来告诉发出请求的线程；比如发出读请求后，线程就直接返回了，待读工作完成后会有回调通知线程读的工作已经完成好了。  

- **事件驱动**就是说，如果可读这个事件发生了，那么就会驱动进行读事件；同样，如果已经可写了，那么就会驱动写事件的发生，这些会涉及到ChannelHandler等，后面会一一介绍的。  

## Netty的核心组件
### 1. Channel
Java NIO中就有Channel这个概念，可以一同理解。它代表一个可以到实体的开放连接（这里的实体可以指一个硬件设备，一个文件，一个套接字等），又可以称作"通道"，是传入（入站）或者传出（出站）数据的载体。

### 2. 回调
一个回调就是一个方法，就是在完成某种操作时执行的方法，当一个回调被触发时，相关的事件会被一个ChannelHandler的实现处理。比如数据准备好可以开始读时，就可以回调进行读操作。  

### 3. Future
Future提供另一种在操作完成时通知应用程序的方式，会在未来某个时刻完成，并提供对其结果的访问，与ChannelFutureListener配合使用。  

### 4. 事件和ChannelHandler
Netty使用不同的事件来通知我们状态的改变或者操作的改变，使得我们可以基于发生的事件来激活适当的动作。这些动作可以是：记录日志、数据转换、流控制、应用程序逻辑、数据的读写等。  

Netty的事件是按照与入站或出站数据流相关性进行分类的。  
Netty定义了两个重要的ChannelHandler子接口，后面还会经常碰到的：  
- ChannelInboundHandler：处理入站数据以及各种状态变化  
- ChannelOutboundHandler：处理出站苏沪并且允许拦截所有的操作

对于入站数据或者相关状态更改而出发的事件有： 
- 连接被激活或者连接失活
- 数据读取
- 用户事件
- 错误事件，期间发生的exception

对于出站事件，包括：
- 打开或者关闭到远程节点的连接
- 将数据写到或者冲刷到套接字

## 小结
本文中，主要简单论述了Netty是什么，几个核心组件，其实Netty中还有几个很重要的概念，EventLoopGroup，EventLoop，ChannelHandler，ChannelHandlerContext，ChannelPipeline等很重要的概念。  

Netty通过触发事件将Selector从应用程序中抽象出来，消除了所有本来需要手动派写的派发代码，比如selector单线程循环判断是否有需要处理的新事件，之后根据事件的具体类型做进一步处理，如果可读就分配线程去进行读操作。后面会写个Java NIO和Netty在写网络通信时代码上的区别对比。  

在Netty内部，会为每个Channel分配一个EventLoop，用来处理所有事件，包括：  
- 注册感兴趣的事件
- 将事件派发给ChannelHandler
- 安排进一步的动作  

需要注意的是，下面这句话可能还会在之后的文章出现，因为很重要。EventLoop本身只由一个线程驱动，该线程处理了一个Channel的所有I/O事件，并且在该EventLoop在整个生命周期内不会改变，这样带来的优越性会在之后的线程模型相关文章中再具体说，而本身其实若干个EventLoop又会放在一个EventLoopGroup中。  

一个Channel的EventLoop是唯一的，但是多个Channel可以指向同一个EventLoop。一个Channel会有一个ChannelPipeline与之绑定，在ChannelPipeline中会被放置一个或多个ChannelHandler，pipeline嘛就是管道，用于放多个handler，handler是干啥的呢，就是用来进行对入站或出站的处理。简单来说，就是对一个Channel的ChannelPipeline中若干个ChannelHandler的调用以及其他一些处理都是通过一个EventLoop来完成的，具体的，会在之后的文章进行更详细的分析。