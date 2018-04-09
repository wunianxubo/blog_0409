---
title: CountDownLatch  
date: 2017-08-15 20:33:00  
tags: [java,java并发]    
categories: java  
---
## 什么是CountDownLatch ##
**CountDownLatch也叫闭锁，在jdk1.5中引入，允许一个或多个线程等待其他线程完成操作后再执行。  
CountDownLatch内部会维护一个初始值等于线程数量的计数器，主线程执行await方法，如果计数器大于0，那么主线程就阻塞等待。每当一个线程完成任务，计数器值就减一。当计数器为0时，表示所有的线程都已经完成任务，阻塞等待的主线程被唤醒，继续执行。**  
![image](http://osrmzp0jr.bkt.clouddn.com/countdownlatch1.png)  
```
public class TestCountDownLatch{
	public static void main(String[] args){
	final CountDownLatch latch = new CountDownLatch(5);
	LatchDemo ld = new LatchDemo(latch);

	long start = System.currentTimeMillis();
	
	for(int i = 0; i<5; i++){
		new Thread(ld).start();
	}
	
	try{
		latch.await();//await方法挂起主线程
	}catch(InterruptedException e){
	}
	
	long end = System.currentTimeMillis();
	
	System.out.println("耗费时间："+(end-start));
	}
}

class LatchDemo implements Runnable{
	private CountDownLatch latch;
	public LatchDemo(CountDownLatch latch){
		this.latch=latch;
	}

	public void run(){
		synchronized(this){
			try{
				for(int i=0; i<50000; i++){
					if(i%2==0){
						System.out.println(i);
					}
				}
			}finally{
				latch.countDown();//计数器减一
			}
		}
	}
}
```
## CountDownLatch的实现原理 ##
CountDownLatch的实现主要是基于Java同步器AQS。其内部维护了一个AQS子类，并重写了相关的方法。  
### await实现
主线程执行await方法，tryAcquireShared方法中，如果state（state为未完成线程任务的线程个数）不等于0，返回-1，将主线程加入到等待队列中，主线程通过LockSupport.park(this)被挂起。  
await():使当前线程在计数器至零之前一直等待，除非线程被中断。  
### countDown实现
countDown方法实现state的减1操作，即通过unsafe.compareAndSwapInt方法设置state值。  
如果state的值为0，那么通过LockSupport.unpark唤醒await方法中挂起的主线程。  

## join方法
**join用于让当前的执行线程等待join线程执行结束。其实现原理是不停检查join线程是否存活，如果join线程存活就让当前线程继续等待下去。** CountDownLatch也实现了join功能，并且比join功能更丰富。