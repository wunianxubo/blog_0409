---
title: Java对象的创建过程
date: 2017-10-11 17:35:00  
tags: [java,java虚拟机]    
categories: 深入理解Java虚拟机  
---
# 一、对象的创建
## 对象创建有以下几个步骤：  
1.虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，必须先执行相应的类加载过程。  
2.在类加载检查通过后，接下来虚拟机为新生对象分配内存。  
3.内存分配后，虚拟机将分配到的内存空间都初始化为零值（不包括对象头）。  
4.执行<init>方法，把对象按照程序员的意愿进行初始化。  
5.将对象在堆上分配的内存地址赋给实例变量。  
## 初始化顺序：  
1、先父类，后子类  
2、先属性，再构造代码块，最后构造函数  
3、先静态，后非静态  
4、同一类型，按代码顺序先后执行  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B.png)  

# 二、对象的内存布局
对象在内存中存储的布局可以分为三块区域：**对象头、实例数据和对齐填充**。  
## 对象头
包括两部分，第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID等。另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。  
## 实例数据
实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是子类中定义的，都需要记录下来（包括private的）。  
## 对齐填充
非必要存在，仅仅起着占位符的作用。  

# 三、对象的访问定位
建立对象是为了使用对象，java程序通过栈上的reference数据来操作堆上的具体对象。主流的访问方式主要有**使用句柄**和**直接指针**两种。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%AF%B9%E8%B1%A1%E8%AE%BF%E9%97%AE%E5%AE%9A%E4%BD%8D.jpg)