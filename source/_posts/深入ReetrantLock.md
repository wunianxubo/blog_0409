---
title: 深入ReetrantLock  
date: 2017-08-21 21:18:04  
tags: [java,java并发]    
categories: java  
---

# 一、Lock接口
在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，在java5之后，并发包中增加了Lock接口（以及相关实现类）用来实现锁功能，它提供了与synchronized类似的功能，但是在使用时需要显式地获取和释放锁。虽然缺少了synchronized隐式获取释放锁的便捷性，但是却拥有锁获取和释放的可操作性、可中断的获取锁、超时获取锁等多种synchronized不具备的同步特性。  
<!-- more -->

```
Lock lock = new ReetrantLock();
Condition condition = lock.newCondition();
lock.lock();
try{
    while(条件判断){
        condition.await();
    }
}finally{
    lock.unlock();
}
```
需要显式的获取锁，并在finally块中显式的释放锁，保证在获取锁之后，最终能够被释放。  
## 1、公平锁与非公平锁的实现  
公平性与否，是针对于获取锁而言的。如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。
### ①非公平锁实现  
对于非公平锁，只要CAS设置同步状态成功，就表示当前线程获取了锁。  
**1、线程A和B同时执行CAS指令，假设线程A成功，线程B失败，表示线程A成功获取锁，并把同步器中的exclusiveOwnerThread设置为线程A。**  
**2、竞争失败的线程B，在nonfairTryAcquire方法中，会再次尝试获取锁，在这段时间如果线程A释放锁，线程B就可以直接获取锁而不用挂起。**  

### ②公平锁实现  
**在公平锁中，每当线程执行lock方法时，如果同步器的队列中有线程等待，则直接加入到队列中。**  
公平锁的实现方法tryAcquire与非公平锁的实现方法nonfairTryAcquire方法比较，唯一不同的是判断条件多了hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，表示有线程比当前线程更早的请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。  

## 2、重入锁实现  
重入锁ReentrantLock，即线程可以重复获取已经持有的锁。通过ReentrantLock的构造函数，还支持选择获取锁时的公平和非公平选择。  
```
if(current == getExclusiveOwnerThread()){
    int nextc = c + acquires;
    if(nextc < 0)
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```
为每个锁关联一个获取计数器和一个所有者线程，当计数值为0时，这个锁就没有被任何线程持有。当线程请求一个未被持有的锁时，JVM会记下锁的持有者，并且将获取计数值置为1，如果同一个线程再次获取这个锁，计数值将递增。每退出一次同步代码块，计数值就递减一次。当计数值为0时，这个锁就被释放。
#  二、Condition接口
任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()、notifyAll()方法，这些方法和synchronized配合，可以实现等待/通知模式。  
Condition接口也提供了类似Object的监视器方法，和Lock配合来实现等待/通知模式。但两者存在差异。  
**1、synchronized中，所有的线程都在同一个object的等待队列上等待；在ReentrantLock中，每个condition都维护了一个等待队列。**  
**2、Condition是与Lock绑定的，所有就有Lock的公平性特性：如果是公平锁，线程按照FIFO的顺序从Condition.await的等待队列中释放；如果是非公平锁，那么后续的锁竞争就不保证FIFO顺序了。**  

### conditon在生产者消费者中的应用场景

```
class ConditionTest{
    final Lock lock = new ReentrantLock();
    final Condition notFull = lock.newCondition();//生产
    final Condition notEmpty = lock.newCondition();//消费
    final Object[] items = new Object[100];//存储商品的容器
    int putptr, takeptr, count;//生产者角标、消费者角标、计数器
    //往容器中存储商品
    public void put(Object x)throws InterruptedException{{
        lock.lock();
        try{
            while(count == items.length)//容器满，生产就等待
                notFull.await();
            items[putptr] = x;
            if(++putptr == items.length)//角标到容器最后，归零
                putptr = 0;
            ++count;
            notEmpty.signal();//唤醒一个消费者
        }finally{
            lock.unlock();
        }
    }
    //从容器出取出商品
    public Object take()throws InterruptedException{{
        lock.lock();
        try{
            while(count == 0)
                notEmpty.await();
            Object x = items[takeptr];
            if(++takeptr == items.length)
                takeptr = 0;
            --count;
            notFull.signal();
            return x;
        }finally{
            lock.unlock();
        }
    }
}
```
## 1、await方法实现
**当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition的signal()方法时，通知当前线程后，当前线程之后从await()方法处返回，并在返回前已经获取到了锁。**  

如果线程A执行await方法：  
1、将线程A加入到conditon的等待队列中，如果最后一个节点的状态是CANCELLED，就从队列中删除。  
2、线程A释放锁，线程A修改AQS的状态state为0，并唤醒AQS同步队列中的线程B，线程B唤醒后，尝试获取锁。  
3、线程A释放锁并唤醒线程B后，如果线程A不在AQS同步队列中，就通过LockSupport.park进行挂起操作。  
4、当线程A被唤醒时，会通过acquireQueued方法竞争锁，如果失败，继续挂起；如果成功，线程A从await位置恢复。  

**acquireQueued方法：节点进入同步队列之后，就进入了一个自旋的过程，每个节点（每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋的过程中退出，否则依旧留在这个自旋的过程中（并会阻塞节点的线程）。** 
如下图所示：  
![image](http://osrmzp0jr.bkt.clouddn.com/d_%E5%89%AF%E6%9C%AC.jpg)

## 2、notify方法实现
如果线程B执行notify方法：
1、接着上述场景，线程B执行signal方法，取出等待队列的第一个非CANCELLED的节点线程，即线程A。遇到CANCELLED线程就需要将其从队列中删除。  
2、通过CAS修改线程A的waitStatus为0，表示该节点已经不是处于等待队列状态，并将A插入到AQS的同步队列中。  
3、唤醒线程A，线程A和别的线程进行锁的竞争。  

## 从队列角度看await方法和notify方法
1、当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。  
![image](http://osrmzp0jr.bkt.clouddn.com/a_%E5%89%AF%E6%9C%AC.jpg)  

2、当调用notify()方法时，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点已到同步队列中。  
![image](http://osrmzp0jr.bkt.clouddn.com/b_%E5%89%AF%E6%9C%AC.jpg)  

# 三、synchronized和lock的区别
**1、用法上：  
synchronized是隐式锁，在需要同步的地方加上，可以加在方法上，也可以加在同步块上；  
lock是显式锁，需要指定起始位置和终止位置，在加锁和解锁出通过lock()和unlock()显式指出，需要在finally中释放锁。  
2、功能上：  
ReentrantLock提供了和内置锁synchronized类似的功能和内存语义。此外，ReentrantLock还提供了更丰富的功能。包括定时的锁等待、公平性、实现非块结构的加锁，对线程的等待和唤醒操作更加灵活，一个ReentrantLock可以有多个Condition实例，所以更有扩展性。  
3、性能上：  
synchronized是托管给JVM执行的，而lock是java写的控制锁的代码。在java1.5中，synchronized是性能低效的，相比之下，lock的性能更高。  
但是到了java1.6，发生了变化。synchronized进行了很多优化，有适应自旋、锁消除、锁粗化、轻量级锁、偏向锁等等，导致在java1.6上，synchronized的性能并不比lock差。  
4、机制上：  
synchronized采用的是悲观锁机制，即线程获得的是独占锁，意味着其他线程只能阻塞来等待线程释放锁；  
而Lock用的是乐观锁方式，乐观锁就是每次不加锁，而是假设没有冲突去完成某项操作，如果因为冲突失败就重试，知道成功为止。乐观锁实现的机制就是CAS操作。**  










