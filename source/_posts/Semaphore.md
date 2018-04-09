---
title: Semaphore  
date: 2017-08-17 20:13:00  
tags: [java,java并发]    
categories: java  
---
## 什么是Semaphore
**Semaphore(信号量)是用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用公共资源。**  
**Semaphore的构造方法Semaphore(int permits)接受一个整形的数字，表示可用的许可证数量。许可证的数量也代表着最大并发数。**  
**1、在访问特定资源时，必须使用acquire()方法来获取一个许可证，如果许可的数量为0，该线程就一直阻塞，直到有可用的许可。**  
**2、访问完资源后，使用release()方法归还许可证。**  
Semaphore和ReetrantLock类似，获取许可可用公平策略或者非公平策略，默认情况下使用非公平策略。  

## 应用场景
Semaphore可用于流量控制，比如数据库连接。如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是在读到内存后，我们还需要存储到数据库中，而这时数据库的连接数如果只有10，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。  

```
public class SemaphoreTest{
    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = Excutors.newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);
    public static void main(String[] args){
        for(int i =0; i<THREAD_COUNT; i++){
            threadPool.execute(new Runnable(){
                @Override
                public void run(){
                    try{
                        //读取文件操作
                        s.acquire();
                        //存储数据操作
                        s.release();
                    }catch(InterruptedException e){
                        
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```

## 实现原理
Semaphore的实现主要基于Java同步器AQS，内部使用state表示许可数量。  
### 非公平策略
#### 1、acquire实现
核心代码如下  

```
final int nonfairTryAcquireShared(int acquires){
    for(;;){
        int available = getState();
        int remaining = available - acquires;
        if(remaining<0 || compareAndSetState(available,remaining)
            return remaining;
    }
}
```
acquires的值默认为1，表示尝试获取一个许可，remaining代表剩余的许可数。  
**如果remaining < 0，表示目前没有剩余可用的许可了，那么当前线程进入AQS中的doAcquireSharedInterruptibly方法等待可用的许可，并且被挂起，直到被唤醒。**  
#### 2、release实现
核心代码如下  

```
protected final boolean tryReleaseShared(int release){
    for(;;){
        int current = getState();
        int next = current + releases;
        if(next < current)
            throw new Error("Maximum permit count exceeded")
        if(compareAndSetState(current,next)
            return true;
    }
}
```
releases的值默认为1，表示尝试释放1个许可；next代表许可释放成功后，可用许可的数量。  
**1、通过unsafe.compareAndSwapInt修改state的值，确保同一时刻只有一个线程可以释放成功。**  
**2、如果许可释放成功，那么当前线程进入到AQS的doReleaseShared方法，唤醒队列中等待许可的线程。**  

#### 非公平性的体现
**当一个线程A执行acquire方法时，会直接尝试获取许可，而不管同一时刻阻塞队列中是否有线程也在等待许可，如果恰好有线程C执行release释放许可，并唤醒阻塞队列中的第一个等待的线程B，这时，线程A和B共同竞争可用许可，不公平性体现出来，线程A没有等待就和线程B同等对待。**   

### 公平策略
#### 1、acquire实现
核心代码如下  
```
final int TryAcquireShared(int acquires){
    for(;;){
        if(hasQueuePredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if(remaining<0 || compareAndSetState(available,remaining)
            return remaining;
    }
}
```
acquires的值默认为1，表示尝试获取一个许可，remaining代表剩余的许可数。  
与非公平策略相比，只是多了一个对阻塞队列的检查。  
**1、如果阻塞队列中没有等待的线程，那么当前线程可以参与许可的竞争；**  
**2、如果阻塞队列中有等待的线程，那么直接插入阻塞队列尾结点并挂起，等待被唤醒。**  
#### 2、release实现
和非公平策略一样。  
