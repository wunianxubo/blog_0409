---
title: IO多路复用select、poll、epoll的区别与使用 
date: 2017-10-27 20:36:00  
tags: [操作系统]    
categories: 操作系统
---
## I/O复用简述
&emsp;I/O多路复用技术是为了解决进程或线程阻塞到某个I/O系统调用而出现的技术，使进程不阻塞于某个特定的I/O系统调用。  
&emsp;select()、poll()、epoll()都是I/O多路复用的机制。**I/O多路复用通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。** select()、poll()、epoll()本质上都是同步IO，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说，读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。  

## 一、select()的使用
```
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
nfds:要监视的文件描述符的范围，一般取描述符数的最大值+1
readfds：监视的可读描述符集合，只要有文件描述符即将进行读操作，这个文件描述符就存储到这。
writefds：监视的可写描述符集合。
exceptfds：监视的错误异常描述符集合。
```
### 功能
&emsp;监视并等待多个文件描述符的属性变化（可读、可写或错误异常）。select()函数监视的文件描述符分为三类，分别是readfds，writefds，exceptfds。调用select()函数会阻塞，直到有描述符就绪（有数据可读、可写或者有错误异常，或者超时），函数才会返回。当select()函数返回后，可以通过遍历fdset来找到就绪的描述符。
### select()的优缺点
**优点：** 几乎在所有平台是都支持，具有良好的跨平台支持。  
**缺点：**  
1、每次调用select()，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大；同时每次调用select()，都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大。  
2、单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是这样也会造成效率的降低。  

## 二、poll()的使用
select()和poll()系统调用的本质一样，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll()没有最大文件描述符数量的限制（但数量过大后性能也会下降）。poll()和select()同样存在一个缺点是，包含大量文件描述符的集合被整体复制于用户态和内核态之间，而不论这些文件描述符是否就绪，随文件描述符数量的增加而线性增大。  
```
int poll(struct pollfd *fds, nfds_t nfds, int timeout)
```
### 功能
&emsp;监视并等待多个文件描述符的属性变化。  
&emsp;poll()的实现和select()非常相似，只是描述fd集合的方式不同，poll()使用pollfd结构而不是select()的fd_set结构，还有就是poll()没有最大文件描述符数量的限制，其他的都差不多。  

## 三、epoll()的使用
epoll()是select()和poll()的增强版本。但epoll更加灵活，没有描述符限制。epoll()使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。  
epoll操作过程需要三个接口：
```
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
### 1、int epoll_create(int size)
**功能：** 该函数生成一个epoll专用的文件描述符（创建一个epoll的句柄）。在创建好epoll句柄后，它就是会占用一个fd值，在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。  
### 2、int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
**功能：** epoll的事件注册函数，它不同于select()是在监听事件时告诉内核要监听什么类型的事件，epoll需要先注册要监听的事件类型。  
### 3、int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
**功能：** 等待事件的发生，收集在epoll监控的事件中已经发送的事件，类似于select()调用。  

&emsp;epoll对文件描述符的操作有两种模式：LT模式和ET模式。LT模式是默认模式，区别如下：  
**LT模式：** 当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。  
**ET模式：** 当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。  
ET模式很大程度上减少了epoll事件被重复触发的次数，因此效率比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。  

&emsp;在select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll()事件通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。  

### epoll()的优点
1、监视的描述符数量不受限制，它所支持的fd上限是最大可以打开文件的数目，一般远大于2048。1GB内存机器上大约是10万左右。  
2、I/O效率不会随着监视fd的数量的增长而下降。select,poll需要不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll也需要调用epoll_wait()不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是epoll在设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait()中睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节约了大量的CPU时间。  
3、select,poll每次调用都要把fd集合从用户态复制到内核态，而epoll只要拷贝一次，这节省了很大的开销。  
