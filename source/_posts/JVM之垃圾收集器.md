---
title: 深入java虚拟机之垃圾收集器
date: 2017-10-11 10:33:00  
tags: [java,java虚拟机]    
categories: 深入理解Java虚拟机  
---
### 七种垃圾收集器
1、Serial（串行GC）-复制  
2、ParNew（并行GC）-复制  
3、Parallel Scavenge（并行回收GC）-复制  
4、Serial Old(MSC)（串行GC）-标记-整理  
5、CMS（并发GC）-标记-清除  
6、Parallel Old（并行GC）-标记-整理  
7、G1（jdk1.7才正式商用）  
其中，1~3用于年轻代垃圾回收（minor GC），4~6用于老年代垃圾回收（full GC），G1独立完成“分代垃圾回收”。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.jpg)  

### 并行与并发
**并行**：多条垃圾回收线程同时操作  
**并发**：垃圾回收线程和用户线程一起操作，但不一定是并行的，可能交替执行。  
### 常用的五种组合
1、Serial/Serial Old  
2、ParNew/Serial Old，与上面相比，只是比年轻代多了多线程垃圾回收而已  
3、ParNew/CMS，当下比较高校的组合  
4、Parallel Scavenge/Parallel Old，自动管理的组合  
5、G1，最先进的收集器，需要jdk1.7update14以上  

## 一、新生代垃圾收集器

### 1、Serial收集器  
![image](http://osrmzp0jr.bkt.clouddn.com/Serial%E6%94%B6%E9%9B%86%E5%99%A8.png)  

是最基础、最古老的收集器，是一个单线程的收集器，是Client模式下的默认收集器。  
**优点**：简单高效（与其他收集器的单线程比），对于限定单个CPU的环境下，Serial收集器由于没有线程交互的开销，专心做垃圾收集可以获得最高的单线程收集效率。  
**缺点**：是单线程处理，并且会stop the world，即在它进行垃圾收集时，必须暂停其他所有的工作线程，这对很多应用来说难以接受的。  
**应用**：Serial收集器对于运行在Client模式下的虚拟机来说是一个很好的选择。  

### 2、ParNew收集器  
![image](http://osrmzp0jr.bkt.clouddn.com/parnew.png)  

其实就是Serial收集器的多线程版本，除了使用多条线程进行垃圾收集之外，其他都与Serial完全一样。  
**特点**：  
1.是并行收集器  
2.是server模式下的首选收集器  
3.可以和CMS收集器相配合，但是Parallel Scavenge与CMS是无法配合的  
4.使用复制算法进行垃圾回收  

### 3、Parallel Scavenge收集器
![image](http://osrmzp0jr.bkt.clouddn.com/Parallel%20Scavenge.png)  

&emsp;Parallel Scavenge的关注点和其他收集器不同，**CMS等收集器的关注点是尽可能的缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge的目的是达到一个可控制的吞吐量**。  
吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间)  
&emsp;停顿时间越短就越适合需要与用户交互的程序，良好的响应速度可以提升用户体验；而**高吞吐量则可以高效率的利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。**  

## 二、老年代垃圾收集器  
### 1、Serial Old收集器
是Serial收集器的老年代版本。  
**特点**：  
1.是一个单线程收集器  
2.使用标记-整理算法  
3.主要在Client模式下使用  
&emsp;在Server模式下，主要有两个用途：一个是在jdk1.5前与Parallel Scavenge配合使用，还有一个是作为CMS的后备预案，在发生Concurrent Mode Failure时使用。  

### 2、Parallel Old收集器
是Parallel Scavenge老年代版本。  
**特点**：  
1.多线程  
2.标记-整理算法  
3.jdk1.6之后提供  
**作用**：在Parallel Old之前，如果新生代选择了Parallel Scavenge，那么老年代只能选择Serial Old，由于Serial Old的拖累，使用整体性能不一定比ParNew/CMS高。直到有了Parallel Old，“吞吐量优先”菜真正有了名副其实的组合。  

### 3、CMS（Concurrent Mark Sweep）收集器  
![image](http://osrmzp0jr.bkt.clouddn.com/CMS.png)  

**是一种追求回收停顿时间最短的收集器。**  是基于“标记-清除”算法实现的。  
**分为四个步骤**：  
**1.初始标记**：仅仅是标记一下GC Roots能直接关联到的对象，速度很快  
**2.并发标记**：进行GC Roots的追踪过程  
**3.重新标记**：为了修正并发标记期间由于用户程序继续运作而导致标记产生变动的那一部分对象的标示记录    
**4.标记清除**  
&emsp;其中，初始标记和重新标记仍然需要“Stop the world”，但是由于整个过程中最耗时的并发标记和并发清除过程是可以与用户线程一起工作的，所以从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。  
**优点**：并发收集，低停顿  
**缺点**：  
1.对cpu资源非常敏感  
2.无法处理浮动垃圾，当CMS运行期间预留的内存无法满足程序需要，会出现“Concurrent Mode Failure”  
3.标记-清除算法，会产生大量的空间碎片  
&emsp;默认条件下，当老年代使用68%即触发GC，1.6中已提高为92%。  

## 三、通用收集器G1（Garbage-First）
![image](http://osrmzp0jr.bkt.clouddn.com/G1.png)  

1.7u14后正式提供商用G1，以前都是测试用的。G1是一款面向服务端应用的垃圾收集器。  
**特点**：  
1.并行与并发  
2.分代收集  
3.空间整合：基于“标记-整理”，不会产生内存空间碎片  
4.可预测的停顿：能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。  
&emsp;G1将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留了新生代和老年代的概念，但是他们并不是物理隔离的了，他们都是一部分Region（不需要连续）的集合。G1跟踪各个Region里面垃圾堆积的价值大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region，这也就是Garbage-First的由来。  
**难题**：对象可能在不同的Region中引用。  
**解决方法**：G1通过Remember Set来避免进行全堆扫描。G1中每个Region都有一个对应的Remember Set，一旦发现一个对象引用了另一个Region的对象，就通过CardTable把相关引用信息记录到被引用对象所属的Region的Remember Set中。当进行GC时，在GC根节点的枚举范围中加入Remember Set，就可以保证不对全堆扫描也不会有遗漏。  
**主要有以下几个步骤**：  
**1.初始标记**：标记一下GC Roots能直接关联到的对象，并修改TAMS的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象  
**2.并发标记**：从GC Roots进行可达性分析，找出存活对象  
**3.最终标记**：为了修正并发标记期间由于用户程序继续运作而导致标记产生变动的那一部分对象的标示记录  
**4.筛选回收**  


