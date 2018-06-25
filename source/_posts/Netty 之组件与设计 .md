---
title: Netty之组件与设计
date: 2018-06-15 21:53:00  
tags: [Netty]    
categories: Netty  
toc: true
---

### 一、Channel、EventLoop、ChannelFuture
这些类合在一起，可以被认为是Netty网络抽象的代表：  
- Channel——Socket
- EventLoop——控制流、多线程处理、并发处理
- ChannelFuture——异步通知
<!-- more -->
#### EventLoop接口
EventLoop定义了Netty的核心抽象，用来处理连接的生命周期中所发生的事件。之后在Netty的线程模型中还会做更详细的讲解，我们先来看看EventLoop、Channel之间是怎样的关系。  

![image](http://osrmzp0jr.bkt.clouddn.com/EventLoopGroup.png)  
![image](https://upload-images.jianshu.io/upload_images/2184951-2e248d85df2a1a86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/490)  
首先有EventLoopGroup，用来管理EventLoop的生命周期，默认情况下，一个EventLoopGroup中有两倍线程的EventLoop。创建Channel后，将Channel注册到EventLoop，之后在Channel的整个生命周期中，都由这个EventLoop来处理I/O事件。而实际上，每个EventLoop都会维护一个Selector和TaskQueue（之后的文章会进一步讨论）。  

EventLoop和Channel一般是一对多的形式，他们之间关系可总结为以下几点：  
- 一个EventLoopGroup包含一个或者多个EventLoop；  
- 一个EventLoop在它的生命周期内只和一个Thread绑定；  
- 与EventLoop绑定的Thread会处理所有EventLoop要处理的I/O事件，也就是注册在EventLoop上Channel需要处理的事件；  
- 一个Channel在它的生命周期内只注册于一个EventLoop；  
- 一个EventLoop可能会被分配给一个或多个Channel。  

#### ChannelFuture
Netty中所有的I/O操作都是异步的，因此一个操作不会立刻返回，我们需要一种在之后的某个时间点确定其结果的方法。Netty提供了ChannelFuture接口，它的addListener()方法注册了一个ChannelFutureListener，用于在某个操作完成时（无论是否成功）得到通知。这个异步是指编程模型上的异步，基于reactor模式的事件驱动，事件处理器的注册和处理器的执行都是异步的。  

如上一篇文章中的代码实例，bind()用于异步的绑定服务器，ctx.write(in)用于异步的将缓存中数据写到context等等。  

### 二、ChannelHandler和ChannelPipeline

#### ChannelHandler
ChannelHandler派生出ChannelInboundHandler和ChannelOutboundHandler接口。  
如果有入站事件被读取，那么它会从ChannelPipeline的头部开始流动，并传递给第一个ChannelInboundHandler。这个handler具体会做怎样的处理，取决于它的具体功能，处理完之后，数据将会被传递给handler链中的下一个ChannelInboundHandler。最终到达ChannelPipeline的尾端，这样所有的入站事件处理就完成了。出站事件的处理与之相反。  

#### ChannelPipeline
![image](https://upload-images.jianshu.io/upload_images/2184951-beacd91367f1f4eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/629)  

ChannelPipeline为ChannelHandler链提供了容器，并且定义了用于在该链上传播入站和出站事件流的API。每个Channel在创建时，会分配一个专属的ChannelPipeline。ChannelHandler安装到ChannelPipeline的过程如下：  
- 一个ChannelInitializer的实现被注册到ServerBootstrap中；
- 当ChannelIntializer.initChannel()方法被调用时，ChannelInitializer将在ChannelPipeline中安装一组自定义的ChannelHandler，这些handler与Channel想关联，也可以根据需要进行动态的添加和删除；
- ChannelIntializer将它自己从ChannelPipeline中移除。  

ChannelHandler的执行顺序是由它们被添加的顺序决定的，第一个执行完会传递给下一个handler。  

#### ChannelHandlerContext
ChannelHandler被添加到ChannelPipeline时，它会被分配一个ChannelHandlerContext，它代表了ChannelHandler和ChannelPipeline之间的绑定，它的主要功能是管理它所关联的ChannelHandler和在同一个ChannelPipeline中的其他ChannelHandler之间的交互。
#### Channel和ChannelHandlerContext相同方法调用的区别：
Channel和ChannelHandlerContext有一些共同的方法，但是它们的调用存在一些差别：
- 如果调用Channel或者ChannelPipeline上的这些方法，它们将沿着整个ChannelPipeline进行传播；
- 而如果调用位于ChannelHandlerContext上的这些相同方法，就会从ctx关联的ChannelHandler开始，并且只会传播给位于该ChannelPipeline中的下一个能够处理这个事件的ChannelHandler。  

### 三、ServerBootstrap和Bootstrap  
Netty的引导类主要为应用程序提供了网络层的配置。对于服务端，涉及到将一个进程绑定到某个指定的端口；对于客户端，涉及到将一个进程连接到另一个运行在某个指定主机的指定端口上的进程。分为ServerBootstrap和Bootstrap这两种引导类。    

这两个是两种类型的引导：一个用于客户端（Bootstrap），一个用于服务器（ServerBootstrap），无论你的应用程序是使用哪种协议或者处理哪种类型的数据，唯一决定它该使用哪种引导类的是它是客户端还是服务端。  

两者最大的区别在于：引导一个客户端只需要一个EventLoopGroup，但是引导一个服务端需要两个EventLoopGroup，这是为什么呢？ 
原因在于服务器需要两组不同的Channel，第一组只包含一个ServerChannel，代表服务器自身的已绑定到某个本地端口的正在监听的套接字Socket；而第二组将包含所有已创建的用来处理传入客户端连接的Channel。  

![image](http://osrmzp0jr.bkt.clouddn.com/evenloop_meitu_2.jpg)  

与ServerChannel相关联的EventLoopGroup将分配一个为传入连接请求创建Channel的EventLoop，一旦连接被接收，第二个EventLoopGroup就会给它的Channel分配一个EventLoop。

