---
title: 原子操作及CAS算法  
date: 2017-08-12 21:48:04  
tags: [java,java并发]    
categories: java  
---
# 一、原子操作的实现原理
## 处理器如何实现原子操作
**1. 使用总线锁保证原子性**  
&emsp;&emsp;**总线锁**就是使用处理器提供的一个LOCK#信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，这样，该处理器就可以独占共享内存。  
**2. 使用缓存锁保证原子性**  
&emsp;&emsp;在同一时刻，我们只需要保证对某个内存地址的操作是原子性即可，但总线锁定把CPU和内存之间的通信锁住了，使得在锁定期间，其他的处理器不能操作其他内存地址的数据，所以总线锁定开销较大，在某些场合用缓存锁代替总线锁。  
&emsp;&emsp;**缓存锁定**是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声言LOCK#信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，**因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。**  
## Java如何实现原子操作  
&emsp;&emsp;Java中可以通过**循环CAS**和**锁**的方式来实现原子操作。  
**1. 使用循环CAS实现原子操**  
&emsp;&emsp;JVM中的CAS操作是利用处理器提供的CMPXCHG指令实现的，而处理器执行CMPXCHG指令是一个原子性操作，所以能保证原子操作的实现。  
**2. 使用锁机制实现原子操作**  
&emsp;&emsp;锁机制保证了只有获得锁的线程才能够操作锁定的内存区域。**除了偏向锁，JVM实现锁的方式都用了循环CAS，即当一个线程想进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块的时候使用循环CAS释放锁。**  

# 二、CAS算法剖析  
&emsp;&emsp;**CAS(Compare And Swap)，即比较并替换。三个操作数：内存值V，预估值A，更新值B，当且仅当V==A时，V=B。CAS的比较和替换是一组原子操作，不会被外部打断，属于硬件级别的操作，效率比加锁高** 

## **AtomicInteger为例：**  

```
public class AtomicInteger extends Number implements java.io.Serializable{
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    static{
        valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"))
    }
    private volatile int value;
    public final int get(){
        return value;
    }
}
```
①**Unsafe**是CAS的核心类，它提供了**硬件级别的原子操作**。  
②**valueOffset**表示变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址来获取数据的原值的。  
③**value**是用**volatile**修饰的，保证了value的可见性。  

AtomicInteger在并发下的累加操作过程：  
&emsp;&emsp;在jdk1.8中，比较和替换操作放在Unsafe类中实现。假设线程A和B同时执行getAndAdd操作：  
1. AtomicInteger中的value值为3，即内存中value为3，根据Java内存模型，线程A和B各持有一份value的副本，值为3.
2. 线程A通过getIntVolatile(var1,var2)方法获取value值为3，此时线程切换，线程A挂起。
3. 线程B通过getIntVolatile(var1,var2)方法获取value值为3，并利用**compareAndSwapInt**方法比较和替换，修改内存值为4，线程切换，线程B挂起。
4. 线程A恢复，利用**compareAndSwapInt**方法比较，发现手中的值3和内存值4不一致，此时value正在被另一个线程修改，线程A不能修改value值。
5. 线程的**compareAndSwapInt**实现，循环判断，重新获取value的值，value为volatile修饰变量，其他线程的修改，线程A总是能看到。直到**compareAndSwapInt**修改成功并返回true。 

&emsp;&emsp;**整个过程中，使用CAS保证了value修改的线程安全性。**  

### compareAndSwapInt方法：

```
public final native boolean compareAndSwapInt(...)
```

&emsp;&emsp;**compareAndSwapInt是一个本地方法的调用，会根据处理器的类型来决定是否为CMPXCHG指令添加lock前缀。**多处理器，为CMPXCHG指令添加lock前缀；单处理器则省略lock前缀（不需为单处理器提供内存屏障效果）。  

### lock前缀：
&emsp;&emsp;lock前缀的指令**在多核处理器下**会引发两件事：
1. **将当前处理器缓存行的数据写回到系统内存。**
2. **这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。**  

## CAS实现原子操作的三大问题
**（1）ABA问题**  
&emsp;&emsp;CAS在操作值的时候，需要检查值有没有发生变化，如果没有变化则更新。但如果一个值原来是A，变成了B，最后又变成A，那么使用CAS检查变化时会发现他的值没有发生变化，而实际上却变化了。  
**解决方法：在变量前面加上版本号，每次变量更新时把版本号加1，那么A->B->C变成1A->2B->3A,在Atomic里提供了一个类AtomicStampedReference**来解决ABA问题。  
**（2）循环时间长开销大**  
&emsp;&emsp;自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。  
**（3）只能保证一个共享变量的原子操作**
&emsp;&emsp;当对一个共享变量执行操作时，可以使用循环CAS的方式保证原子操作，但对多个共享变量操作时，循环CAS就不能保证操作的原子性，这时候可以用锁。**jdk提供AtomicReference保证了引用之间的原子性，就可以把多个变量放在一个对象里进行CAS操作。**