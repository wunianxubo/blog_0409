---
title: Java内存溢出与内存泄漏
date: 2017-12-08 10:23:00  
tags: [java,java虚拟机]    
categories: 深入理解Java虚拟机  
---
## 一、为什么要了解内存泄露和内存溢出？
1、内存泄露一般是代码设计存在缺陷导致的，通过了解内存泄露的场景，可以避免不必要的内存溢出和提高自己的代码编写水平；  
2、通过了解内存溢出的几种常见情况，可以在出现内存溢出的时候快速的定位问题的位置，缩短解决故障的时间。  
<!-- more -->
## 二、基本概念
**内存泄露**：无用对象持续占有内存或无用对象的内存得不到及时释放，从而造成的内存空间的浪费称为内存泄漏。  
**内存溢出**：指程序运行过程中无法申请到足够的内存而导致的一种错误。内存溢出通常发生于OLD段（老年代）或Perm段（永久代）垃圾回收后，仍然无内存空间容纳新的Java对象的情况。  
从定义上可以看出内存泄露是内存溢出的一种诱因，不是唯一因素。  

## 三、Java中主要的内存泄漏场景
- 静态集合类，容器中的对象在程序结束之前都不会释放。这些静态变量的生命周期和应用程序一致，他们所引用的对象Object也不能被释放，因为他们也将一直被Vector引用。
```
static Vector v = new Vector();
for(int i = 1; i<100; i++){
    Object o = new Object();
    v.add(o);
    o = null;
}
```
- 修改hashset中对象的参数值，且参数是计算哈希值的字段。  
    当一个对象被存储进HashSet集合中以后，就不能修改这个对象中的那些参与计算哈希值的字段，否则对象修改后的哈希值与最初存储进HashSet集合中时的哈希值就不同了，在这种情况下，即使在contains方法使用该对象的当前引用作为参数去HashSet集合中检索对象，也将返回找不到对象的结果，这也会导致无法从HashSet集合中删除当前对象，造成内存泄露。
- 数据库连接、网络连接、IO连接，不再使用时需要close方法释放
- 监听器，用到的监听器在释放的时候没有去删除，从而增加了内存泄漏的可能性
- 变量不合理的作用域，成员变量在使用后依然存在（结局：将变量设为局部变量）
- 单例造成内存泄漏，单例对象在初始化后将在JVM的整个生命周期内存在，如果外部对象（生命周期比较短）持有该引用，那么该外部对象就不能被回收，从而导致内存泄漏

## 四、内存溢出的几种情况：
### 1、堆内存溢出（outOfMemoryError：java heap space）
&nbsp;&nbsp;分为两种情况：一种是堆内存确实不够，还有一种是由于内存的泄漏而造成的内存溢出。  
&nbsp;&nbsp;在jvm规范中，堆中的内存是用来生成对象实例和数组的。我们只要不断的创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，当对象数量达到最大堆容量限制后产生内存溢出异常。    如果细分，堆内存还可以分为年轻代和年老代，年轻代包括一个eden区和两个survivor区。  
![image](http://osrmzp0jr.bkt.clouddn.com/20141107224401036.png)  
当生成新对象时，内存的申请过程如下：  
a、jvm先尝试在eden区分配新建对象所需的内存；  
b、如果内存大小足够，申请结束，否则下一步；  
c、jvm启动Minor GC，试图将eden区中不活跃的对象释放掉，释放后若Eden空间仍然不足以放入新对象，则试图将部分Eden中活跃对象放入Survivor区；如果整个新生代都放不下对象时，可以直接在老年代分配内存；    
d、Survivor区被用来作为Eden及old的中间交换区域，当OLD区空间足够时，Survivor区的对象会被移到Old区，否则会被保留在Survivor区；  
e、当OLD区空间不够时，JVM会在OLD区进行full GC；  
f、full GC后，若Survivor及OLD区仍然无法存放从Eden复制过来的部分对象，导致JVM无法在Eden区为新对象创建内存区域，则出现”out of memory错误”。  
```
public class MemoryLeak {
   
    private String[] s = new String[1000];
 
    public static void main(String[] args) throws InterruptedException {
        Map<String,Object> m =new HashMap<String,Object>();
        int i =0;
        int j=10000;
        while(true){
            for(;i<j;i++){
                MemoryLeak memoryLeak = new MemoryLeak();
                m.put(String.valueOf(i), memoryLeak);
            }
        }
    }
}
```
#### 解决办法：  
&nbsp;&nbsp;出现这种异常，一般手段是先通过内存映像分析工具(如Eclipse Memory Analyzer)对dump出来的堆转存快照进行分析，重点是确认内存中的对象是否是必要的，先分清是因为内存泄漏(Memory Leak)还是内存溢出(Memory Overflow)。  
&nbsp;&nbsp;如果是内存泄漏，可进一步通过工具（如Jrockit等工具）查看泄漏对象到GC Roots的引用链。于是就能找到泄漏对象时通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收。
如果不存在泄漏，那就应该检查虚拟机的参数(-Xmx与-Xms)的设置是否适当。

### 2、方法区内存溢出（outOfMemoryError：permgem space）
在jvm规范中，方法区主要存放的是类相关信息（如类名、访问修饰符、常量池、字段描述、方法描述等），静态变量，常量，即时编译器编译后的代码等。  
所以**如果程序加载的类过多，或者使用反射、gclib等这种动态代理生成类的技术，就可能导致该区发生内存溢出。**  在经常动态生成大量Class的应用中，要注意这点。    
```
jvm参数：-XX:PermSize=2m -XX:MaxPermSize=2m
将方法区的大小设置很低即可，在启动加载类库时就会出现内存不足的情况
```
### 3、线程栈溢出（java.lang.StackOverflowError）
线程栈时线程独有的一块内存结构，所以线程栈发生问题必定是某个线程运行时产生的错误。  
**一般线程栈溢出是由于递归太深或方法调用层级过多导致的。**  
```
public class StackOverflowTest{

public static void main(String[] args){
    int i =0;
    digui(i);
}

private static void digui(int i){
    System.out.println(i++);
    String[] s = new String[50];
    digui(i);
    }
}
```
## 五、为了避免内存泄露，在编写代码的过程中可以参考下面的建议：
1、尽早释放无用对象的引用  
2、使用字符串处理，避免使用String，应大量使用StringBuffer，每一个String对象都得独立占用内存一块区域  
3、尽量少用静态变量，因为静态变量存放在永久代（方法区），永久代基本不参与垃圾回收  
4、避免在循环中创建对象  
5、开启大型文件或从数据库一次拿了太多的数据很容易造成内存溢出，所以在这些地方要大概计算一下数据量的最大值是多少，并且设定所需最小及最大的内存空间值。  
