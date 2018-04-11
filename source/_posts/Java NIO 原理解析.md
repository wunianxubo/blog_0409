---
title: Java NIO 原理解析
date: 2018-04-11 17:05:00  
tags: [java,java基础]    
categories: java基础  
---
## 前言
NIO（Non-blocking I/O，在Java领域，也成为New I/O），是一种同步非阻塞的I/O模型，也是I/O多路复用的基础，Netty其实就是对NIO的一种封装，以实现高性能。  
那么NIO的本质是怎样的？它是怎样与事件模型结合来解放线程、提高系统吞吐的呢？  
<!-- more -->
## 传统BIO模型分析
<pre>
{
 ExecutorService executor = Excutors.newFixedThreadPollExecutor(100);//线程池

 ServerSocket serverSocket = new ServerSocket();
 serverSocket.bind(8088);
 while(!Thread.currentThread.isInturrupted()){//主线程死循环等待新连接到来
 Socket socket = serverSocket.accept();
 executor.submit(new ConnectIOnHandler(socket));//为新的连接创建新的线程
}

class ConnectIOnHandler extends Thread{
    private Socket socket;
    public ConnectIOnHandler(Socket socket){
       this.socket = socket;
    }
    public void run(){
      while(!Thread.currentThread.isInturrupted()&&!socket.isClosed()){死循环处理读写事件
          String someThing = socket.read()....//读取数据
          if(someThing!=null){
             ......//处理数据
             socket.write()....//写数据
          }
      }
    }
}
</pre>
这是一个经典的一个连接一个线程的模型，主线程死循环等待新连接的到来，到来后accept阻塞等待数据的到来，到来后便开启新的线程来处理数据。  

这里使用线程池，来让线程的创建和回收成本相对较低。在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的I/O并且编程模型简单，也不用过多考虑系统的过载、限流等问题。  

但是，这个模型最本质的问题在于，严重依赖于线程。但是线程是很昂贵的资源，主要体现在：
1. 线程的创建和销毁成本很高，在Linux这样的操作系统中，线程本质上就是一个进程。创建和销毁都是重量级的系统函数。
2. 线程本身占用较大内存，像Java的线程栈，一般至少分配512K～1M的空间，如果系统中的线程数过千，恐怕整个JVM的内存都会被吃掉一半。
3. 线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统load偏高、CPU 使用率特别高（超过20%以上)，导致系统几乎陷入不可用的状态。
4. 容易造成锯齿状的系统负载。因为系统负载是用活动线程数或CPU核心数，一旦线程数量高但外部网络环境不是很稳定，就很容易造成大量请求的结果同时返回，激活大量阻塞线程从而使系统负载压力过大。  

所以，在面对十万甚至百万级连接的时候，传统的BIO模型是无能为力的，我们需要更高效的I/O处理模型。  
## 常见I/O模型对比
所有的系统I/O都分为两个阶段：等待就绪和操作。比方说，读函数就分为等待系统可读和真正的读；写函数与之类似。  

需要说明的是等待就绪的阻塞是不使用CPU的，是在**空等**。而真正的读写操作的阻塞是使用CPU的，真正在干活，而且这个过程非常快，属于memory copy，带宽通常在1GB/s以上，可以理解为基本不耗时。  

下图是几种常见I/O模型的对比：  
![image](http://osrmzp0jr.bkt.clouddn.com/io%20moxing.jpg)  

以socket.read()为例：  

对于BIO：如果TCP RecvBuffer里没有数据，函数会一直阻塞，直到收到数据，返回读到的数据。  

对于NIO：如果TCP RecvBuffer里有数据，就把数据从网卡读到内存，并且返回给用户；反之则直接返回0，不会造成阻塞。  

对于AIO：不但等待就绪时非阻塞的，就连数据从网卡到内存的过程都是异步的，由操作系统去完成一系列的操作。  

换句话说，BIO中用户最关心"我要读"，NIO中用户最关心"我可以读了"，AIO中用户更关注的是"读完了"。

NIO的重要特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作时同步阻塞的（消耗CPU但是性能非常高）。  

## 如何结合事件模型使用NIO同步非阻塞特性
<pre>
interface ChannelHandler{
      void channelReadable(Channel channel);
      void channelWritable(Channel channel);
   }
   class Channel{
     Socket socket;
     Event event;//读，写或者连接
   }

   //IO线程主循环:
   class IoThread extends Thread{
   public void run(){
   Channel channel;
   while(channel=Selector.select()){//选择就绪的事件和对应的连接
      if(channel.event==accept){
         registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器
      }
      if(channel.event==write){
         getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件
      }
      if(channel.event==read){
          getChannelHandler(channel).channelReadable(channel);//如果可以读，则执行读事件
      }
    }
   }
   Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器
  }
</pre>

由上面的示例可以看出NIO是怎样解决掉线程的瓶颈并处理连接的：NIO由原来的阻塞读写（占用线程）变成了单线程轮询事件，找到可以进行读写的网络描述符进行读写。除了事件的轮询是阻塞的（只需要单线程），剩余的I/O操作都是纯CPU操作，没有必要开启多线程。  

单线程处理I/O的效率确实很高，没有线程的切换，只是拼命的读、写、选择事件。但现在的服务器都是多核服务器，充分利用多核心进行I/O，无疑对效率有更大提升。  

我们需要的线程，主要包括以下几种：  
1. 事件分发起，单线程选择就绪的事件。  
2. I/O处理器，包括connect、read、write等，这种纯CPU操作，一般开启CPU核心个线程就可以。  
3. 业务线程，在处理完I/O后，业务还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等，只要有阻塞，就需要单独的线程。  

![image](http://osrmzp0jr.bkt.clouddn.com/reactor.jpg)

## Proactor与Reactor
一般情况下，I/O复用机制都需要事件分发器（event dispatcher）。事件分发器的作用是，将那些读写事件源分发给各读写事件的处理者（handler）。就像快递到了之后在楼下喊：谁谁谁的快递到了，快来拿吧！  

开发人员在开始的时候需要在分发器那里注册感兴趣的事件，并提供相应的处理者或者回调函数；事件分发器在适当的时候，会将请求的事件分发给这些handler或者回调函数。  

涉及到事件分发器的两种模式是：Proactor和Reactor。Reactor是基于同步I/O的，而Proactor是和异步I/O相关的。在Reactor模式中，事件分发器等待某个事件发生，就会把事件传给事先注册的事件处理函数或回调函数，由后者来做实际的读写操作。  

而在Proactor模式中，事件处理者（或者由事件分发器发起）直接发起一个异步读写操作，而实际的工作是由操作系统完成的。事件分发器得知了这个请求，它默默等待这个请求的完成，然后转发完成事件给后续相应的事件处理者或回调。这种异步模式基于操作系统底层异步API，我们可称之为"系统级别"或者真正意义上的异步，因为具体的读写都是由操作系统代劳的。

### 在Reactor中实现读
- 注册读就绪事件和相应的事件处理器。
- 事件分发器等待事件。
- 事件到来，激活分发器，分发器调用事件对应的处理器。
- 事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。

### 在Proactor中实现读：
- 处理器发起异步读操作（注意：操作系统必须支持异步IO）。在这种情况下，处理器无视IO就绪事件，它关注的是完成事件。
- 事件分发器等待操作完成事件。在分发器等待过程中，操作系统利用并行的内核线程执行实际的读操作，并将结果数据存入用户自定义缓冲区，最后通知事件分发器读操作完成。
- 事件分发器呼唤处理器。事件处理器处理用户自定义缓冲区中的数据，然后启动一个新的异步操作，并将控制权返回事件分发器。  

通俗易懂来谈两者区别：  
reactor：能收了你跟俺说一声。  
proactor: 你给我收十个字节，收好了跟俺说一声。  

## Buffer的选择
通常情况下，操作系统的一次写操作分为两步：  

1. 将数据从用户空间拷贝到系统空间。  
1. 从系统空间往网卡写。同理，读操作也分为两步：  
    ① 将数据从网卡拷贝到系统空间；  
    ② 将数据从系统空间拷贝到用户空间。    

对于NIO来说，缓存的使用可以使用DirectByteBuffer和HeapByteBuffer。如果使用了DirectByteBuffer，一般来说可以减少一次系统空间到用户空间的拷贝。但Buffer创建和销毁的成本更高，更不宜维护，通常会用内存池来提高性能。  

如果数据量比较小的中小应用情况下，可以考虑使用heapBuffer；反之可以用directBuffer。  

两者区别：  
DirectBuffer（直接缓冲区）：缓冲区建立在物理内存中，可提高效率。  
HeapBuffer：将缓冲区建立在JVM的内存中。  

## NIO存在的问题
使用NIO != 高性能，当连接数<1000，并发程度不高或者局域网环境下NIO并没有显著的性能优势。  

NIO并没有完全屏蔽平台差异，它仍然是基于各个操作系统的I/O系统实现的，差异仍然存在。使用NIO做网络编程构建事件驱动模型并不容易，陷阱重重。  

推荐大家使用成熟的NIO框架，如Netty，MINA等。解决了很多NIO的陷阱，并屏蔽了操作系统的差异，有较好的性能和编程模型。  

## 参考文章
Java NIO浅析 https://zhuanlan.zhihu.com/p/23488863