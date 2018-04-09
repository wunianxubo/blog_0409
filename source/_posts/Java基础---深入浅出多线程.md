---
title: 深入浅出多线程
date: 2017-07-08 17:12:55
tags: [Java,多线程] 
categories: Java

---
## **1、多线程的引入**
  多线程的相关内容是Java基础中非常重要的一部分，这两天对这部分知识进行了梳理，以达到复习和查漏补缺的目的。
  首先，多线程指的是在单个程序中可以同时运行多个不同的线程执行不同的任务，多线程编程的目的其实就是“最大限度地利用CPU资源”，我们接下来介绍下进程/程序/线程之间有什么区别和联系。

 **1. 进程与程序的区别是什么呢？**
 （1）程序是长期存在的，进程是暂时的，是程序在数据集上的一次运行，有创建有撤销，存在是暂时的
 （2）程序是静态的观念，进程是动态的观念
 （3）进程具有并发性，而程序没有
 （4） 进程是竞争计算机资源的基本单位，程序不是
 （5）进程和程序不是一一对应的，一个程序可对应多个进程即多个进程可执行同一程序，一个进程可以执行一个或多个程序
 **2. 进程和线程又有什么区别呢？**
 线程是指进程内的一个执行单元，也是进程内的可调度实体。与进程的区别：
 （1）调度：线程作为调度和分配的基本单位，进程作为拥有资源的基本单位
 （2）并发性：不仅进程之间可以并发执行，同一个进程的多个线程之间也可以并发执行
 （3）拥有资源：进程是拥有资源的一个独立单位，线程不拥有系统资源，但可以访问隶属于进程的资源
 （4）系统开销：在创建和撤销进程时，由于系统都要为之分配和回收资源，导致系统的开销明显大于创建和撤销线程时的开销
    
## **2、创建线程的两种方式**
### **继承Thread类**

 - 定义一个类去继承Thread
 - 重写run方法
 - 创建子类对象，就是创建线程对象
 - 调用start方法，开启线程并让线程执行，同时告诉jvm去调用run方法
 
 几个小问题： 
 **1.  线程对象调用run()方法和调用start()方法有什么区别**
   调用run()方法不开启线程，仅仅是线程调用方法。调用start()方法开启线程，并让jvm调用run方法，在开启的线程中执行。
 **2. 为什么要继承Thread类？**
   因为Thread类描述线程事务，具备线程应有功能。
 **3. 为什么不直接创建Thread类的对象呢？**
  这么做start()调用的是Thread类中的run方法，此方法内部为空，不做任何事情，没有我们需要让线程执行的代码。
 
    多线程执行时，在栈内存中，其实每一个执行线程都有一片自己所属的栈内存空间，进行方法的压栈和弹栈。当执行线程的任务结束了，线程自动在栈内存中释放；当所有的执行线程都结束了，进程也就结束了。

### **实现Runnable接口**

 - 定义一个类来实现Runnable接口
 - 覆盖接口中的run方法，将线程任务代码定义到run方法中
 - 创建Thread类的对象（只有创建Thread对象，才能创建线程。）
 - 将Runnable接口的子类对象作为参数传递给Thread类的构造函数
 - 调用Thread类的start()方法开启线程
 
Thread源码调用：

```
class Thread{
	private Runnable target;
	Thread(Runnable target){
		this.target = target;
		}
	public void run(){
		if(target != null){
			target.run();
		}
	public void start(){
			run();
		}
	}
}

Runnable d = new Demo();
Thread t = new Thread(d);
t.start();
```

**1. 为什么将Runnable接口的子类对象作为参数传递给Thread类的构造函数？**
   结合上面的Thread源码，我们不难看出，因为线程任务已被封装到Runnable接口的run()方法中，而这个run方法所属于Runnable接口的子类对象，所以将这个子类对象作为参数传递给Thread的构造函数，这样，线程对象创建时就可以明确要执行的线程任务了。
   
**2.  实现Runnable接口的方式较继承Thread的方式有何优势？**
  1、实现Runnable接口避免了单继承的局限性
  2、实现Runnable接口的方式，更加的符合面向对象，线程分为两部分，一部分是线程对象，一部分是线程任务。
  继承Thread类：线程对象和线程任务耦合在一起，一旦创建Thread类的子类对象，既是线程对象，又有线程任务
  实现Runnable接口：将线程任务单独分离出来，封装成对象，类型就是Runnable接口类型，Runnable接口对线程对象和线程任务进行了解耦

## **3、多线程的安全问题**
  线程安全问题产生的原因：
  **1、多个线程在操作共享的数据**
  **2、线程任务操作共享数据的代码有多条（有多次运算）**
  解决思路：只要让一个线程在执行线程任务时，将多条操作共享数据的代码执行完，在执行过程中，不要让其他线程参与运算。

使用同步synchronized：
```
synchronized(锁对象){

	//需要被同步的代码...

}
```
  考虑到大家对synchronized锁可能很难理解，我做个比较形象的比喻来方便大家理解：使用同步时会用到锁，这就好比是**在火车上上厕所**。这又从何说起呢？
   **火车上有个人想上厕所时，会去拿锁进入厕所并关上门，此时其他人拿不到锁是进不了厕所的。当这个人上完厕所后，离开厕所的同时释放了锁，那么其他人都具有同等的机会来竞相拿到锁进入厕所。**这就好比一个线程拿到锁，此时其他线程无法进入同步代码块或者同步函数执行代码，当一个线程执行完后释放锁，其他线程争相获得cpu的执行权，以此来获得锁进入同步代码块执行代码。希望这个例子能给大家一些启发。


同步代码块的好处：解决了多线程的安全问题
同步弊端：降低了程序的性能（很多线程访问，每次访问都需要判断锁，做了很多无用功）；当线程任务中出现了多个同步（多个锁）时，如果同步嵌套了其他的同步，这时候容易引发死锁。
**同步前提：必须保证多个线程在同步中使用的是同一个锁。（也以此前提来判断同步是否书写正确）**

**同步代码块与同步函数的区别？**
1、同步函数使用的锁是固定的this；同步代码块使用的锁可以是任意对象
2、当线程任务只需要一个同步时，完全可以使用同步函数；当线程任务中需要多个同步时，必须通过锁来区分，这时候必须使用同步代码块
注：static同步函数使用的锁不是this，而是字节码文件对象，类名.class

## **4、多线程间的通信**
多线程间最为常见的应用案例：生产者消费者问题，具体说来就是：生产和消费同时进行，需要多线程，但是执行的任务却不相同，处理的资源确实相同的。在生产者生产了商品后应该告诉消费者进行消费，这时的生产者需要处于等待状态；消费者在消费了商品后应该告诉生产者进行生产，这时消费者需要处于等待状态。

  **生产一个消费一个的情况**：生产者生产面包，判断盘子中是否有面包，有的话生产者进入等待状态；没有的话将生产后的面包放于盘子中，唤醒消费者进行消费。如果盘子中没有面包，消费者就进入等待状态；如果盘子中有面包，那么消费者消费，同时唤醒生产者进行生产。以下为具体代码实现：
  
```
class Resource{
	private String name;
	private int count = 1;
	//定义标记flag
	private boolean flag = false;
    //生产者生产行为
	public synchronized void set(String name){
		//如果flag为true,说明盘子里有面包，不需要生产，生产者进入等待模式。
		if(flag)
			this.wait();
		this.name = name + count;
		count++;
		System.out.println("...生产者..."+this.name);
		//完成生产，将标记改为true
		flag = true;
		//唤醒消费者
		this.notify();	
	}
	//消费者消费行为
	public synchronized void out(){
		//如果flag为false,说明盘子里没有面包，消费不了，消费者者进入等待模式。
		if(!flag)
			this.wait();
		System.out.println("...消费者..."+this.name);
		//完成消费，盘子里没有面包了，将标记改为false
		flag = false;
		//唤醒生产者
		this.notify();
	}
}

//描述生产者
class Producer implements Runnable{
	private Resource r;
	Producer(Resource r){
		this.r = r;
	}
	public void run(){
		while(true){
			r.set("面包");
		}
	}
}

//描述消费者
class Consumer implements Runnable{
	private Resource r;
	Consumer(Resource r){
	this.r = r;
	}
	public void run(){
		r.out();
		}
	}

public class Tset1{
	public static void main(String[] args){
		Resource r = new resource();
		Producer pro = new Producer(r);
		Consumer con = new Consumer(r);
		Thread t1 = new Thread(pro);
		Thread t2 = new Thread(con);
		t1.start();
		t2.start();
	}
}
```

### **等待/唤醒机制**
wait()：会让线程处于等待状态，其实就是将线程临时存储到线程池中。
notify()：会让线程池中任意一个等待的线程唤醒。
notifyAll()：会唤醒线程池中所有的等待线程。
**记住：这些方法必须使用在同步中，因为必须要标识wait、notify等方法所属的锁，同一个锁上的notify只能唤醒该锁上被wait的线程。**
  
**多生产多消费的形式**：多个生产者，多个消费者的情况，如果延用上面的代码则会遇到一下几个问题，具体大家可以自己实施下，无非是，多new几个生产者、消费者，多创建几个线程。

***问题一：部分生产了的商品没有被消费，同一个商品可能被消费多次***
原因：被唤醒的线程没有再次判断标记，造成问题的发生
解决：只要让被唤醒的线程重新再次去判断标记就可以了，将if判断标记的方式改为while判断标记的方式。
**记住：多生产多消费，必须是while判断语句。**

***问题二：改为while后，死锁了***
原因：生产方唤醒了线程池中生产方的线程（由于唤醒的是线程池中**任意一个**线程），唤醒后判断标记flag，发现为true进入等待状态，此时所有线程都进入等待状态，程序无法继续执行，死锁发生。
解决：希望本方唤醒对方，没有对应的方法，只能使用notifyAll的方法。

经过上面两步，我们解决了遇到的问题，但是这种方式最大的问题是，**效率相对比较低**，那我们有没有更好的解决方法呢？答案是肯定的！

### **Lock接口**
   Lock接口提供了更加面向对象的锁，在所中提供了更加显示的锁操作，我们可以通过lock()方法来获得锁，也可以用过unlock()方法来达到释放锁的目的，比同步更加厉害，可以用来替代synchronized。
   新锁（Lock）替代旧锁（synchronized），那么旧锁上的监视器方法（wait, notify, notifyAll）也在新锁上得到替换（await,  signal, signalAll）。在jdk1.5中，将这些原有的监视器方法封装到了一个Condition对象中，想要换取监视器的方法，就需要通过lock的newCondition方法获取Condition对象。通过使用新锁，我们可以在一个锁上创建多个监视器对象。
  下图为Lock与synchronized的对比：
  
  
  ![Lock与synchronized对比](http://img.blog.csdn.net/20170320101442665?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3VuaWFuXzkzMDEyNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


下面我们使用新锁的方式，介绍一个**生产多个消费多个**的问题：一边在生产商品，将生产的放于容器中；另一边从容器中取商品消费。


   ![多个消费多个生产模型](http://img.blog.csdn.net/20170320102244052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3VuaWFuXzkzMDEyNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```
class Test2{
	final Lock lock = new ReentranLock();//锁
	final Condition notFull = lock.newCondition();//生产
	final Condition notEmpty = lock.newCondition();//消费
	final Object[] items = new Object[20];//存储商品的容器
	int putptr, takeptr, count;//生产者角标、消费者角标、计数器
//往容器中存储商品
public void put(Object x)throws InterruptException{
	lock.lock();//加锁
	try{
		while(count == items.length)//判断计数器是否已到数组长度，满了生产就进入等待状态
			notFull.await();
		items[putptr] = x;//按照角标存储商品
		if(++putptr == items.length)//存储角标到达数组长度，角标归零，重新从头存储商品
			putptr = 0;
		++count;//计数器自增
		notEmpty.signal();//唤醒消费者
		}finally{
			lock.unlock();//释放锁
		}
	}
public Object take()throws InterruptException{
	lock.lock();
	try{
		while(count==0)
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

### **线程的状态**
线程状态图如下所示：
![线程状态图](http://img.blog.csdn.net/20170320105528467?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3VuaWFuXzkzMDEyNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## **5、多线程的细节**

**1. sleep方法和wait方法的异同？**
相同点：都可以让线程处于冻结状态
不同点：
1，sleep必须指定时间；wait可指定时间，也可不指定时间
2，sleep时间到，线程处于临时阻塞或进行状态；wait如果没有时间，必须要通过notify或者notifyAll唤醒
3，sleep不一定非要定义在同步中；wait必须要定义在同步中
4，都定义在同步中时，线程执行到sleep不会释放锁；线程执行到wait会释放锁

**2. 线程如何停止呢？**
线程结束就是让线程任务执行完，run方法结束。在run方法中通常都定义循环，只要控制住循环就可以了。

```
class Test3 implements Runnable{
	private boolean flag = true;
	public void run(){
	while(flag)
		System.out.println(Thread.currentThread().getName()+"-------->");
	} 

	//对标记修改的方法
	public void changeFlag(){
		flag = false;
	}
}

class StopThreadDemo{
	public static void main(String[] args){
		Test3 t = new Test3();
		Thread t1 = new Thread(t);
		Thread t2 = new Thread(t);
		t1.start();
		t2.start();
		int x = 0;
		//满足循环要求，改变标记使其他线程任务能够结束，同时break跳出循环，让主线程也可以结束。
		while(true){
			if(++x = 50){
			t.changeFlag();
			break;
			}
			System.out.println("main---->"+x);
		}
		System.out.println("over");
	}
}
```
注意：万一线程在任务中被冻结了，那么它还能去判断标记吗？不能！
解决：如果目标线程等待很长时间，则应该使用interrupt方法来中断该等待，所谓的中断并不是停止线程。
  interrupt的功能是将线程的状态清除，让线程恢复到运行状态（让线程重新具备cpu的执行资格）。由于是强制性的，所有会有异常发生，可以在catch中捕获异常，在异常处理中，改变标记让循环结束，让run方法结束。

**3.  守护线程的概念**
守护线程为后台线程，一般创建的都是前台线程。
相同点：前台、后台线程运行时都是一样的获取cpu的执行权和执行资格，都可以通过run方法结束，线程结束的方式结束。
不同点：当进程中所有的前台进程都结束了，无论后台处于什么样的状态，都会结束，从而进程会结束，进程的结束都是依赖于前台进程。

**4.  线程的优先级**
用数字标识。1-10，其中默认的初始优先级是5，最明显的优先级是1，5，10。

**5.  线程组ThreadGroup**
可以通过Thread的构造函数明确新线程对象所属的线程组。线程组的好处是，可以对多个同组线程进行统一的操作，效率高，默认是都属于main线程组。

**6.  jion()方法和yield()方法**
Thread1.jion()：主线程执行到这里，知道Thread1线程要加入执行，主线程释放了执行权、执行资格，并处于冻结状态。什么时候能恢复呢？等Thread1线程执行完后。
Thread2.yield()：线程临时暂停，将执行权释放，让其他线程有机会获得cpu的执行权。

**7. 开发中，线程的匿名内部类体现**
第一种方式：

```
new Thread(){
	public void run(){
		for(int x=0; x<40; x++){
		System.out.println(Thread.currentThread().getName()+ "...X..." +x);
		}
	}
}.start();
```
第二种方式：

```
Runnable r = new Runnable(){
	public void run(){
		for(int x=0; x<40; x++){
		System.out.println(Thread.currentThread().getName()+ "...Y..." +x);
	}
};
new Thread(r).start();
```

**8.  关于成员变量与局部变量**
如果一个变量是成员变量，那么多个线程对同一个对象的成员变量进行操作时，他们对该成员变量是彼此影响的，也就是说一个线程对成员变量的改变会影响到另一个线程。如果一个变量是局部变量，那么每个线程都会有一个该局部变量的拷贝，一个线程对该局部变量的改变不会影响到其他的线程。

  **先总结这么多，刚买了《Java并发编程实战》，后面肯定要通过好好读这本书对并发有个更好的理解，大家多多交流，一起努力！**