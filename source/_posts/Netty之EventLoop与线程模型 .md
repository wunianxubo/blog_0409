---
title: Netty之EventLoop与线程模型
date: 2018-06-16 21:53:00  
tags: [Netty]    
categories: Netty  
toc: true
---

### 前言
首先，线程模型指定了操作系统、编程语言或者应用程序的*上下文中的线程管理*的关键方面。Netty的线程模型，强大而又易用，在简化代码的同时，最大限度的提高性能和可维护性。学习这部分可能需要有一定的多线程的知识的基础，可以看看《Java并发编程实战》或者《并发编程的艺术》这两本书。
<!-- more -->

### 线程模型概述
在这里，我们先看看常见的线程模型是怎样的，然后我们再看看Netty的线程模型如何，看看他们各自的优缺点。  

早期Java语言中，我们使用多线程处理的主要方式是按需创建和启动新的Thread来执行并发的任务——在高负载下性能很差，需要重复的创建和销毁线程。  
#### 线程池
所以，在Java5之后引入了Executor API，使用线程池通过缓存和重用Thread极大地提高了性能。线程池的具体原理可参考我之前的文章[《Java中的线程池》](http://xiaonanbobo.com/2017/08/30/Java%E4%B8%AD%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%B1%A0/)。线程池的工作模式如下：  
- 从池的空闲线程列表中选择一个Thread，并且指派它去运行一个已提交的任务。（一个Runnable的实现）
- 当任务完成时，将该Thread返回给该列表，使其可被重用。  

*虽然池化和重用线程相对于每次都创建和销毁线程是一种进步，但是它并不能消除由上下文切换所带来的开销，随着线程数量的增加很快变得明显，并且会越来越严重。*  

#### 上下文切换
上下文切换：CPU通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。但是，在切换前会保存上一个任务的状态，以便下次切换回这个任务时，可以再加载这个任务的状态。所以任务从保存到再加载的过程就是一次上下文切换。

### EventLoop
运行任务来处理在连接的生命周期内发生的事件是网络框架的基本功能。与之对应的编程上的构造称为事件循环——在Netty中通过EventLoop来适配，一个EventLoop对应一个Selector和一个TaskQueue。  
![image](https://upload-images.jianshu.io/upload_images/2184951-2e248d85df2a1a86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/490)   

```
while(!terminated){
    //阻塞，直到有事件已经就绪可以被执行
    List<Runnable> readyEvents = blockUntilEventReady();
    //循环遍历，并处理所有的事件
    for(Runnable ev : readyEvents){
        ev.run();   
    }
}
```
在这个模型中，一个EventLoop将由一个不会改变的Thread驱动，同时任务（Runnable或Callable）可以直接提交给EventLoop实现，以立即执行或者调度执行。根据配置或者CPU核心数的不同，可能会创建多个EventLoop实例用以优化资源的使用，并且EventLoop与Channel之间可以是一对多的关系，即多个Channel公用一个EventLoop。  

在Netty4中，Channel所有的I/O操作和事件都交给分配给了EventLoop的那个不会变的Thread来处理。由于至始至终都在同一个线程中进行处理，这样也避免了一些线程安全的问题。  

### 实现细节
#### 线程管理
Netty线程模型的优越性能取决于对于当前执行的Thread的身份的确定，也就是确定当前执行的Thread是不是分配给当前Channel以及其对应的EventLoop的那一个线程（EventLopp与一个Thread至始至终绑定）。  

如果当前调用线程正好是支撑EventLoop的线程，那么所提交的代码块就会被该线程直接执行。否则，EventLoop将调度该任务以便稍后执行，并将它放入到内部队列中（上图的taskQueue）。当EventLoop下次处理它的事件时，它会执行队列中的那些任务。  

#### EventLoop/线程的分配
服务于Channel的I/O事件的EventLoop包含在EventLoopGroup中，根据不同的传输实现，EventLoop的分配方式也不同。  
##### 1.非阻塞传输
异步传输实现只使用了少量的EventLoop以及他们相关联的Thread，而且它们可能被多个Channel共享。这使得可以通过少量的Thread来支撑大量的Channel，而不是每个Channel分配一个Thread。  

![image](http://osrmzp0jr.bkt.clouddn.com/1eventloop_meitu_2.jpg)  

###### 2.阻塞传输
会为每一个Channel分配一个EventLoop，和非阻塞`传输一样的是，每个Channel的I/O事件只会被与之对应的EventLoop的那个Thread所处理。  
