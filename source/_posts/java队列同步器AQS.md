---
title: Java队列同步器AQS  
date: 2017-08-15 16:37:00  
tags: [java,java并发]    
categories: java  
---
## 简述
在java.util.concurrent.locks包中有很多Lock的实现类，**常用的有ReetrantLock、ReadWriteLock，以及CountDownLatch，内部实现都依赖AbstractQueuedSynchronizer（AQS）类，AQS又称为队列同步器，完成代码块的并发访问控制。**  
## 定义

```
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable{
    //等待队列的头结点
    private transient volatile Node head;
    //等待队列的尾结点
    priivate transient volatile Node tail;
    //同步状态
    private volatile int state;
    protected final int getState(){ return state; }
    ...
}
```
队列同步器AQS是用来构建锁或其他同步组建的基础框架。**内部使用一个int类型的成员变量state表示同步的状态，当state=0时，表示没有线程占有锁；state=1时，表示锁已经被占有。** 同步状态state、头结点head、尾结点tail，都是由volatile修饰的，保证了线程之间的可见性。  

### Node结点

```
static final class Node{
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    volatile int waitStatus;//描述对应线程的状态
    volatile Node prev;//上一个节点
    volatile Node next;//后一个节点
    volatile Thread thread;//当前线程
    Node nextwaiter;
    ...
}
```
内部的FIFO队列（先进先出）：  
![image](http://osrmzp0jr.bkt.clouddn.com/aqs1.png)  
黄色结点是head结点，是一个空节点，也就是代表当前持有锁的线程。每当有线程竞争失败，都会插入到队列的尾节点，tail节点始终指向队列的最后一个元素。  
每个节点，除了存储了当前线程，前后节点的引用外，还有一个waitStatus变量，用于描述对应线程的状态：有的线程可能获取到锁，有的可能因为某些原因放弃竞争，有的线程可能在等待满足条件，满足后再执行。共四个状态：  
**1、CANCELLED 取消状态  1  
2、SIGNAL 等待触发状态  -1  
3、CONDITION 等待条件状态  -2  
4、PROPAGATE 状态需要向后传播  -3**  
等待队列是FIFO先进先出，**只有当前一个节点的状态是SIGNAL时，当前节点的线程才能被挂起。**  

## 实现原理
子类重写tryAcquire和tryRelease方法，并通过CAS指令修改状态变量state。  
### 线程获取锁的过程
如线程A和线程B进行竞争：  
1、如果线程A执行CAS指令修改state成功，那么state被修改并返回true，线程A继续执行。  
2、如果线程A执行CAS指令修改state失败，说明线程B此时执行CAS指令成功了，这时线程A会执行步骤3。  
3、生成新Node节点node，通过CAS指令插入到等待队列的队尾（因为在同一时刻可能会有多个Node节点插入到等待队列中）。  
4、**node插入到队尾后，线程不会立马被挂起，而是会进行自旋操作。因为在node的插入过程中，线程B可能已经执行完成，所以要判断该node的前一个节点prev是否为head节点（代表线程B），如果prev==head，说明当前节点是队列中的第一个有效的节点，因此会再次尝试tryAcquire获取锁：**  
&emsp;&emsp;①、如果成功获取到锁，表明线程B已经执行完成，线程A不需要挂起；  
&emsp;&emsp;②、如果获取锁失败，表示线程B还未完成，至少state值还未被修改，那么执行步骤5。  
5、只有在前一个节点的的状态是SIGNAL时，当前节点的线程才能被挂起，那么：  
&emsp;&emsp;**①如果prev的waitStatus=0，那么当前线程通过CAS指令修改waitStatus为Node.SIGNAL；**  
&emsp;&emsp;**②如果prev的waitStatus>0，表明prev的线程状态是CANCELLED，需要从队列中删除；**  
&emsp;&emsp;**③如果prev的waitStatus为Node.SIGNAL，那么通过LockSupport.park()方法把线程A挂起，并等待被唤醒，被唤醒后进入步骤6。**  
6、线程每次被唤醒时，都要进行中断检测，如果发现当前线程被中断，就抛出InterruptedException并退出循环。并不是被唤醒的线程就一定能获得锁，必须调用tryAcquire重新竞争。  

### 线程释放锁的过程
**1、如果头节点head的waitStatus的值为-1，则用CAS指令重置为0；  
2、找到waitStatus的值小于0的节点s，通过LockSupport.unpark(s.thread)唤醒线程。**  

