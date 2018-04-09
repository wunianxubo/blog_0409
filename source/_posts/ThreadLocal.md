---
title: ThreadLocal  
date: 2017-08-23 19:03:04  
tags: [java,java并发]    
categories: java  
---
# 一、ThreadLocal简述
**ThreadLocal，又叫线程本地变量，ThreadLocal变量为每个线程都创建了一个副本，每个线程都可以访问自己内部的副本变量，不同线程之间互不干扰。**  

# 二、深入解析ThreadLocall类  
## 实现原理  
**从线程Thread的角度来看，每个线程内部都会持有一个对ThreadLocalMap实例的引用threadLocals，而ThreadLocalMap实例相当于线程的局部变量空间。引用threadLocals是用来存储实际的变量副本的，键值（key）是当前的ThreadLocal变量，value为变量副本（即ThreadLocal类型的变量值）。**  
![image](http://osrmzp0jr.bkt.clouddn.com/threadLocal.png)  

## ThreadLocalMap
内部使用table数组来存储Entry，默认的大小为INITIAL_CAPACITY(16)，其中两个参数：  
1、size：table中元素的数量，其实就是ThreadLocal变量的数量，因为一个ThreadLocal变量对应table数组中的一个Entry。  
2、threshold：等于table大小的2/3，当size>=threshold时，遍历table并删除key为null的元素；如果删除后size>=threshold的3/4，需要对table进行扩容。  

## ThreadLocal类提供的几个方法

```
public T get(){ }
public void set(T value){ }
public void remove(){ }
protected T initiailValue(){ }
```
set(T value)方法用来设置当前线程中变量的副本；get()方法是用来获取ThreadLocal在当前线程中保存的变量副本；remove()用来移除当前线程中变量的副本。  

### 1、ThreadLocal.set(T value)实现

```
public void set(T value){
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);//从当前线程获取ThreadLocalMap实例
    if(map != null)
        map.set(this, value);//将ThreadLocal实例和value封装成Entry存入ThreadLocalMap中
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t){
    return t.threadLocals;
}
```
从上看出：  
**1、从当前线程获取ThreadLocalMap实例  
2、将ThreadLocal实例和value封装成Entry存入ThreadLocalMap中**  

**map.set(this, value)的内部实现：**  
```
private void set(ThreadLocal<?> key, Object value){
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    
    for(Entry e = tab[i]; e != null; e = tab[i = nextIndex(i,len)]){
        ThreadLocal<?> k = e.get();
        if(k == key){
            e.value = value;
            return;
        }
        if(k == null){
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if(!cleanSomeSlots(i,sz) && sz >= threshold)
        rehash();
}
```
1、通过key.threadLocalHashCode方法生成hash值；  
2、通过 hash & (len-1)，定位到table的位置i，假设table中位置i的元素为a；  
3、如果a != null，假设a中的ThreadLocal的引用为k（即Entry的key）；  
①如果引用k的实例和当前ThreadLocal实例一致，那么修改value值，返回  
②如果引用k为null，说明元素a是陈旧的元素，删除并插入新元素，返回  
③否则通过nextIndex方法找到下一个元素f，继续步骤3  
4、如果a == null，那么把Entry加入到table的位置i上；  
5、通过cleanSomeSlots删除陈旧的元素，如果table中没有元素删除，需要判断当前情况下是否要进行扩容

### 2、ThreadLocal.get()实现

```
public T get(){
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap();//获取当前线程的threadLocals引用
    if(map != null){
        ThreadLocalMap.Entry e = map.getEntry(this);//根据this找到对应的Entry,this为当前ThreadLocal
        if(e != null){
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private Entry getEntry(ThreadLocal<?> key){
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if(e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private T setInitialValue(){
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if(map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
获取当前线程的threadLocals引用：  
1、如果threadLocals不为null，那么通过ThreadLocalMap.getEntry(this)方法找到对应的entry，返回e.value  
2、如果threadLocals为null，那么通过setInitialValue方法初始化，并返回  

### 3、table扩容  
**当table中的元素数量达到阈值threshold的3/4时，会进行扩容操作：**  
```
private void resize(){
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    
    for(int j = 0; j < oldLen; ++j){
        Entry e = oldTab[j];
        if(e != null){
            ThreadLocal<?> k = e.get();
            if(k == nunll){
                e.value = null;
            }else{
                int h = k.threadLocalHashCode & (newLen - 1);
                while(newTab[h] != null)
                    h = nextIndex(h,newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
1、新建新的数组newTab，大小为原来的2倍  
2、复制table的元素到newTab，忽略陈旧的元素，假设table中的元素需要复制到newTab的位置i上，如果位置i存在元素，那么找下一个空位置进行插入  

# 三、应用场景
ThreadLocal不是用来解决对象共享访问的问题的，而主要是**提供了保持对象的方法和避免参数传递的一种便捷的访问对象的方式：**  
**1、每个线程中都有一个自己的ThreadLocalMap类对象，可以将线程自己的对象保存在其中，自己管自己的，线程可以正确的访问到自己的对象**  
**2、将一个共有的ThreadLocal静态实例作为Key，将不同对象的引用保存到不同线程的ThreadLocalMap中，然后在线程执行的各处通过这个静态ThreadLocal实例的get()方法取出自己线程保存的哪个对象，避免了将这个对象作为参数传递的麻烦。**  

最常用的ThreadLocal使用场景是用来解决**数据库连接、Session管理**等，如：  
```
private static final ThreadLocal threadSession = new ThreadLocal();
public static Session getSession() throws InfrastructureException{
    Session s = (Session) threadSession.get();
    try{
        if(s == null){
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    }catch(HibernateException ex){
        throw new  InfrastructureException(ex);
    }
    return s;
}
```
可以看到，在方法getSession()中，首先通过threadSession.get()判断当前线程有没有放进去session，如果还没有，那么通过getSessionFactory().openSession()来创建一个session，再把session用set放进去，实际上是放到当前线程的ThreadLocalMap中。

