---
title: linux的五种I/O模型 
date: 2017-10-27 20:36:00  
tags: [操作系统]    
categories: 操作系统
---
## linux的五种I/O模型：  
1、阻塞I/O  
2、非阻塞I/O  
3、I/O复用  
4、信号驱动I/O  
5、异步I/O  
前面四种都是同步I/O，只有最后一种是异步I/O。  

### 阻塞I/O模型
**进程会一直阻塞，直到数据拷贝完成。** 应用程序调用一个IO函数，导致应用程序阻塞，等待数据准备好，一直等待... 数据准备好了，从内核空间拷贝到用户空间，IO函数返回成功提示。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E9%98%BB%E5%A1%9EIO.png)  

### 非阻塞I/O模型
通过进程反复调用IO函数（多次系统调用，并马上返回），在数据拷贝的过程中，进程是阻塞的。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E9%9D%9E%E9%98%BB%E5%A1%9Eio.jpg)  

### I/O复用模型
主要是select和epoll，对一个IO端口，两次调用，两次返回。能实现同时对多个IO端口进行监听。  

![image](http://osrmzp0jr.bkt.clouddn.com/io%E5%A4%8D%E7%94%A8.jpg)  

### 信号驱动I/O模型
也是两次调用，两次返回。首先允许套接口进行信号驱动IO，并安装一个信号处理函数，进程继续运行并不阻塞。当数据准备好时，进程会收到一个SIGIO信号，可以在信号处理函数中调用I/O操作函数处理数据。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E4%BF%A1%E5%8F%B7%E9%A9%B1%E5%8A%A8io.jpg)  

### 异步I/O模型
**数据拷贝的时候进程无需阻塞**。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的部件在完成后，通过状态、通知和回调来通知调用者的输入输出操作。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%BC%82%E6%AD%A5io.jpg)  

## 五种I/O模型的比较

![image](http://osrmzp0jr.bkt.clouddn.com/io%E6%A8%A1%E5%9E%8B%E6%AF%94%E8%BE%83.jpg)  


 
