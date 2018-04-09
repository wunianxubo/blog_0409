---
title: CyclicBarrier  
date: 2017-08-16 21:34:00  
tags: [java,java并发]    
categories: java  
---
## 什么是CyclicBarrier
CyclicBarrier也叫同步屏障，在jdk1.5中引入，**可以让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，所有被阻塞的线程才能继续执行。**  
CyclicBarrier就好比是一扇门，默认情况是关闭的，堵住所有线程执行的路径，只有所有线程都到达时，门才会打开。  
### 构造方法
1、**默认的构造方法是CyclicBarrier(int parties)，参数parties表示屏障拦截的线程数量，每个线程会调用await方法来告诉CyclicBarrier它已经到达了屏障，然后当前线程就被阻塞。**  
2、**更高级的构造函数是CyclicBarrier(int parties, Runnable barrierAction)用于在所有线程到达屏障时，优先执行barrierAction。**  
![image](http://osrmzp0jr.bkt.clouddn.com/CyclicBarrier.png)  

## CyclicBarrier的实现原理
CyclicBarrier的实现主要基于ReentrantLock。
```
public class CyclicBarrier{
    private static class Generation{
        boolean broken = false;
    }
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();
    ...
}
```
其中，**Generation用来控制屏障的循环使用，如果generation.broken为true的话，说明这个屏障已经损坏了，当某个线程await的时候，直接抛出异常。**  

### await实现原理
**1、每当线程执行await方法时，内部变量count就减1，如果count!=0，说明有线程还没有到达屏障处，那么在锁的条件变量trip上等待。**  
**2、当count==0时，说明所有的线程都到达屏障处，执行条件变量trip的signAll方法来唤醒等待的线程。**  
其中nextGeneration可以实现屏障的循环使用：可以实现重新生成Generation对象和恢复count值。  

## CountDownLatch和CyclicBarrier的区别
**1、CountDownLatch的计数器只可以使用一次；而CyclicBarrier的计数器可以使用reset()方法重置，能够重复利用。**  
**2、CountDownLatch是减计数器方式，构造方法中参数是要等待的线程的个数，每完成一个任务，调用countDown减1，当计数器为0时，说明所有线程的任务已经执行完，阻塞等待的线程被唤醒继续执行；CyclicBarrier中，构造方法中参数是屏障拦截的线程数量，当所有线程到达时，线程数等于parties变量指定的数，栅栏打开，所有线程通过。**  
**3、CountDownLatch中，调用countDown方法实现计数器减1，调用await()方法只进行阻塞，对计数没有影响；CyclicBarrier中，调用await()方法计数器加1，加1后的值没有到达屏障拦截的数量，那么线程被阻塞。**