---
title: 深入剖析volatile  
date: 2018-04-10 10:54:00  
tags: [java,java并发]    
categories: java  
---

# 一、volatile的性质 #
## 1、volatile保证可见性 ##
1. volatile关键字保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这个新值对于其他线程来说是立即可见的。  

<pre> 
//线程1
boolean stop =false;
while(!stop)
  doSomething();

//线程2
stop=true;  
</pre>

上面的代码，可能导致无法中断线程。当线程2更改了stop变量的值后，还没来得及写入内存中，线程2就转去做其他事情了，线程1由于不知道线程2对stop变量的修改，会一直循环下去。  
&nbsp;&nbsp;使用volatile修饰后，发生变化：  
第一：使用volatile会强制将修改的值写入主内存。  
第二：使用volatile的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效。  
第三：由于线程1的工作内存中的缓存变量stop的缓存行无效，所以线程1再次读取变量stop的值时会去主内存读取。    
<!-- more -->
## 2、volatile保证有序性  
<pre> 
//x、y不是volatile变量
//flag为volatile变量
x=2;            //语句1
y=0;            //语句2
flag=true;      //语句3
x=4;            //语句4
y=-1            //语句5
</pre>

①由于flag为volatile变量，在进行指令重排序时，不会将语句3放到语句1、2前面，也不会将语句3放到语句4、5的后面，但是语句1、2的顺序，语句4、5的执行顺序是不做任何保证的。  
②并且volatile保证：执行到语句3时，语句1、2必定是执行完毕类的，而且语句1、2的执行结果对语句3、4、5是可见的。  

## 3、volatile不能保证原子性
&emsp;&emsp;原子性：即一个操作或多个操作要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。  
&emsp;&emsp;可以通过synchronized或Lock进行加锁，来保证操作的原子性。也可以通过使用java.util.concurrent.atomic包下提供的原子操作类（对基本数据类型的一些操作进行了封装）来实现。  
&emsp;&emsp;atomic是利用CAS（Compare And Swap）来实现原子性操作的，CAS实际上利用处理器提供的CMPXCHG指令实现的，而处理器执行CMPXCHG指令是一个原子性的操作。  

## 4、volatile的应用场景
&emsp;&emsp;相较于synchronized，volatile是较为轻量级的同步策略，使用和执行成本更低，因为它不会引起线程上下文的切换和调度。但volatile无法代替synchronized，因为volatile无法保证操作的原子性。  
&emsp;&emsp;使用volatile的两个条件：  
1、对变量的写操作不依赖于当前值  
2、 该变量没有包含在具有其他变量的不变式中  

# 二、volatile的实现原理
## 1、可见性
&emsp;&emsp;处理器为了提高处理速度，不直接和内存进行通讯，而是将系统内存的数据读到内部缓存后再进行操作，但操作完后不知道何时才会写到内存中。  
&emsp;&emsp;对声明了volatile变量进行写操作时，**JVM会向处理器发送一条Lock前缀的指令，将这个变量所在的缓存行的数据写回到系统内存。** 这一步确保了如果有其他线程对声明了volatile的变量进行修改时，则立即更新主内存中的数据。  
&emsp;&emsp;但此时其他处理器的缓存的数据还是旧数据，**所以在多处理器的环境下，为了保证各个处理器的缓存一致，每个处理器会通过嗅探在总线上传播的数据来检查自己的缓存是否过期。当处理器发现自己缓存行对应的内存地址被修改了，就会将当前处理器的缓存行设置为无效状态，当处理器要对这个数据进行修改时，会强制重新从系统内存把数据读到处理器缓存里。** 这一步确保了其他线程获得的声明了volatile的变量都是从主内存中获取的最新的。  
## 2、有序性
&emsp;&emsp;Lock前缀指令实际上相当于一个内存屏障（也称内存栅栏），它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障之后的位置，即在执行到内存屏障这句指令时，在它之前的操作已经全部完成。  

## 3、Lock前缀指令
在多处理器下会引发两件事：  
①将当前处理器缓存行的数据写回到系统内存。  
②这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。  
## volatile的内存语义
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-10%20%E4%B8%8A%E5%8D%8810.32.06.png)
- 第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。  
- 第一个操作时volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。  
- 当第一个操作时volatile写，第二个操作时volatile读时，不能重排序。
### 内存屏障类型
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-10%20%E4%B8%8A%E5%8D%8810.42.48.png)
为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。基于保守策略的JMM内存屏障插入策略如下（**在实际的执行中，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略一些不必要的屏障**）：  
- 在每个volatile写操作的前面插入一个StoreStore屏障，用于禁止上面的普通写和下面的volatile写重排序。
- 在每个volatile写操作的后面插入一个StoreLoad屏障，用于禁止上面的volatile写操作和下面可能有的volatile读/写重排序。
![image](http://osrmzp0jr.bkt.clouddn.com/volatile%E5%86%99.png)
- 在每个volatile读操作的后面插入一个LoadLoad屏障，用于禁止下面的普通读操作和上面的volatile读重排序。
- 在每个volatile读操作的后面插入一个LoadStore屏障，用于禁止下面的普通写操作和上面的volatile读重排序。
![image](http://osrmzp0jr.bkt.clouddn.com/volatile%E8%AF%BB.png)

### volatile写-读的内存语义
```
class VolatileExample{
    int a = 0;
    volatile boolean flag = false;
    
    public void writer(){
        a = 1;        //1
        flag = true;  //2
    }
    
    public void reader(){
        if(flag){      //3
            int i = a; //4
        }
    }
}
```
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-10%20%E4%B8%8A%E5%8D%889.48.12.png)
- volatile写：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。（包含volatile变量flag和定义在flag之前的共享变量a）  
- volatile读：当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量。
综合来看的话，即读线程B读一个volatile变量后，写线程A在写这个volatile变量之前所有可见的共享变量的值都将立即变得对读线程B可见。
### 总结：
- 线程A写一个volatile变量，实际上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所做修改的）消息。
- 线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。
- 线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-10%20%E4%B8%8A%E5%8D%889.48.27.png)  

## 4、JSR-133为什么要增强volatile的内存语义
在JSR-133之前的JMM中，虽然不允许volatile变量之间重排序，但旧的JMM允许volatile变量与普通变量重排序。则上面VolatileExample程序可能被重排序成下列时序运行。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-10%20%E4%B8%8A%E5%8D%8810.13.36.png)  
&emsp;&emsp;此时，步骤3和4重排序，造成结果：读线程B执行4时，不一定能看到写线程A在执行1时对共享变量的修改。而正确的结果应该是线程A对volatile的写，使得共享变量和volatile变量刷新到主内存，之后步骤3对volatile的读使得线程B本地内存数据失效，并从主内存中取出最新数据。  
&emsp;&emsp;在旧的JMM中，volatile的写-读不具有锁的释放-获取所具有的内存语义
。（**volatile写对应锁的释放，volatile读对应锁的获取**），所以专家组决定增强volatile的内存语义，严格限制编程器和处理器对volatile变量与普通变量的重排序，让volatile的写-读具有锁的释放-获取所具有的内存语义。  
# 三、volatile和synchronized的区别 #
1、volatile的本质实在告诉jvm当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中去获取。synchronized则是锁定当前线程才可以访问该变量，其他线程被阻塞住；  
2、volatile只能使用在变量，synchronized可以使用在变量和方法上；  
3、volatile只能实现变量的修改可见性，synchronized可以保证变量修改的可见性和原子性；  
4、volatile不会造成线程的阻塞，synchronized会造成线程的阻塞。