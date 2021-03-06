---
title: 深入java虚拟机之垃圾回收算法
date: 2017-10-10 21:06:00  
tags: [java,java虚拟机]    
categories: 深入理解Java虚拟机  
---
# 一、判断对象是否已死
## 1、引用计数算法
给对象添加一个引用计数器，每当有一个地方引用它时，计数器就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。  
缺点：难以解决循环引用的问题。  
```
ReferenceCountingGC objA = new ReferenceCountingGC();
ReferenceCountingGC objB = new ReferenceCountingGC();
objA.instance=objB;
objB.instance=objA;

objA=null;
objB=null;
```
## 2、可达性分析算法
通过一系列称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，说明该对象是不可用的。  
**可作为GC Roots的对象为：**  
①虚拟机栈中引用的对象  
②方法区中类静态属性引用的对象  
③方法区中常量引用的对象  
④本地方法栈中Native方法引用的对象  
## 3、引用
**强引用（Strong Reference）**：指在程序代码中普遍存在的，类似A a = new A()这样的引用，只要强引用还存在，垃圾回收器永远不会回收掉被引用的对象。  
**软引用（Soft Reference）**：用来描述一些还有用但并非必需的对象。对于软引用关联的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。  
**弱引用（Weak Reference）**：用来描述非必需对象的，但是它的强度比软引用更弱一些，被软引用关联的对象只能生存到下一次垃圾收集之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被软引用关联的对象。  
**虚引用（Phantom Reference）**：称为幽灵引用，是最弱的一种引用关系。一个对象是否有虚引用的存在，不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。对一个对象设置虚引用关联的唯一目的是用来跟踪对象被垃圾回收的状态。  
## 4、finalize自救
即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，要真正宣告一个对象死亡，需要经历两次标记过程：  
1、如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选条件是此对象是否有必要执行finalize方法。当对象没有覆盖finalize方法，或者finalize方法已经被虚拟机调用过，虚拟机将这两种情况视为“没有必要执行”，这时对象宣告死亡。  
2、如果对象被判定有必要执行finalize，那么该对象会放置在一个F-Queue的队列中，并在稍后由一个虚拟机自动建立的、低优先级的Finalizer线程去执行。finalize方法是对象逃脱死亡命运的最后一次机会，稍后GC会对F-Queue中的对象进行第二次小规模的标记，如果对象在finalize()中成功拯救自己——只要重新与引用链的任何一个对象建立连接即可，譬如把自己（this）赋值给某个类变量或者对象的成员变量，那么第二次标记时它将被移除“即将回收”的集合；否则，它就会被真的回收了。  
注意：任何一个对象的finalize方法都只会被系统自动调用一次。  

# 二、垃圾回收算法  
## 1、标记-清除算法
分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。  
有两点不足之处：  
①一个是效率问题，标记和清除两个过程的效率都不高；  
②另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。  
## 2、复制算法
### ①普通的复制算法
将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这块内存用完了，就将还存活着的对象复制到另一块上面，然后再把已使用过的内存一次清理掉。  
这样每次都是对整个半区进行内存回收，不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。  
代价：内存使用率只有原来的一半。  
### ②改进的复制算法
不是按照1:1的比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间。在回收时，会把Eden和其中一个Survivor中还存活的对象一次性放到另一个空的Survivor中，然后清除之前的Eden和第一个Survivor。为了区分，一般称第一个为From Survivor，第二个为To Survivor。他们的比例为8:1:1，只会浪费10%的空间。  
内存担保：有可能存活的对象大于10%，一个Survivor放不下，这时需要有其他内存来作担保，这里会以老年代作为担保。如果放不下，那么通过分配担保机制直接进入老年代。  
## 3、标记-整理算法
老年代没有其他内存给它做担保，所以不能使用复制算法。先标记需要回收的对象，然后把活着的对象向一端进行移动，使他们在物理上连续，然后把边界右侧的内存直接清理掉。  
没有内存碎片，但是效率会稍微低一点，因为要移动对象。  
## 4、分代收集算法
根据对象存活周期的不同将内存划分为几块，一般把Java堆分为**新生代**和**老年代**，根据年代的特点采用最适当的收集算法。  
在**新生代**中，每次垃圾收集时都有大批对象死去，只有少量存活，选用“复制算法”，只需要付出少量存活对象的复制成本就可以完成收集。  
而在**老年代**中，对象存活率高、没有额外空间对它进行分配担保，就使用“标记-清除”或者“标记-整理”算法来进行回收。  

# 三、新生代、老年代和永久代
java虚拟机垃圾收集器关注的内存结构如下：  
![image](http://osrmzp0jr.bkt.clouddn.com/20141107224401036.png)  
堆大小=新生代+老年代，新生代与老年代的比例为1：2，新生代细分为一块较大的Eden空间和两块较小的Survivor空间，分别被命名为from和to。  
## 1、新生代
新创建的对象一般都在新生代中分配内存空间，新生代采用“复制算法”进行垃圾回收，可见上面改进的复制算法。  
java虚拟机对新生代的垃圾回收称为**Minor GC**，次数比较频繁，每次的回收时间较短。使用虚拟机的-Xmn参数可以指定新生代内存大小。  
## 2、老年代
老年代中的对象一般都是长生命周期对象，对象的存活率比较高。当新生代垃圾收集器回收几次之后仍然存活的对象会被移动到老年代内存中（默认是15岁），当大对象无法在新生代找到足够的连续内存时也会直接在老年代中创建。  
老年代中使用“标记-清除”或者“标记-整理”算法进行垃圾回收。java虚拟机对老年代的回收称为**MajorGC/Full GC**，次数相对比较少，每次回收的时间也比较长。  
在新生代中没有足够空间为对象创建分配内存，老年代中内存回收也无法回收到足够的内存空间，并且新生代和老年代都无法扩展时，堆就会产生OutOfMemoryError异常。虚拟机-Xms参数指定最小内存大小，-Xmx参数指定最大内存大小，与新生代大小参数-Xmn之差可计算出老年代最小和最大容量。  
## 3、永久代
永久代指的是虚拟机内存中的方法区，被各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。永久代垃圾回收比较少，效率也比较低，但也必须进行垃圾回收，否则永久代内存不够用时仍然会抛出OutOfMemoryError异常。永久代也使用“标记-清除”或者“标记-整理”算法进行垃圾回收。虚拟机参数-XX：PermSize和-XX：MaxPermSize可以设置永久代的初始大小和最大容量。  



