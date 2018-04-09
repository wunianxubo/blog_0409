---
title: JDK动态代理  
date: 2017-08-25 11:47:04  
tags: [java，java基础]    
categories: java  
---
# 一、静态代理
![image](http://osrmzp0jr.bkt.clouddn.com/%E9%9D%99%E6%80%81%E4%BB%A3%E7%90%86.png)  
1、RealSubject是委托类，Proxy是代理类  
2、Subject是委托类和代理类的接口  
3、request()是委托类和代理类的共同方法  
代码如下：  
```
//接口
interface Subject{
    abstract void request();
}
//委托类
class RealSubject implements Subject{
    public void request(){
        System.out.println("RealSubject");
    }
}
//代理类
class Proxy implements Subject{
    private Subject subject;
    public Proxy(Subject subject){
        this.subject = subject;
    }
    public void request(){
        System.out.println("start");//前置通知
        subject.request();
        System.out.println("stop");//后置处理
    }
}

public class ProxyTest{
    public static void main(String[] args){
        RealSubject subject = new RealSubject();
        Proxy p = new Proxy(subject);//代理类要传入委托类作为参数
        p.request();
    }
}
```
**静态代理**实现中，一个委托类对应一个代理类，**代理类在编译期间就已经确定了**。

# 二、动态代理
动态代理中，必须让代理类和目标类实现相同的接口，客户端通过代理类来调用目标类的方法，**代理类会将所有的方法调用分派到目标对象上反射执行**，在分派的过程中，还可以添加“前置通知”和“后置处理”（如在调用目标方法前校验权限，在调用完目标方法后打印日志等）等功能。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E4%BB%A3%E7%90%86.png)  
相比于静态代理，**动态代理不需要编写各个繁琐的静态代理类**，只需要简单地指定一组接口及目标类对象就可以动态的获得代理对象了。  
## 1、定义业务逻辑  
```
//接口
public interface Service{
    //目标方法
    public abstract void add();
}
//委托类
public class UserServiceImpl implements Service{
    public void add(){
        System.out.println("This is add service");
    }
}
```
## 2、利用java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口定义代理类的实现  
```
class MyInvocationHandler implements InvocationHandler{
    private Object target;
    
    public MyInvocationHandler(Object target){
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)throws Throwable{
        System.out.println("------before--------");
        Object result = method.invoke(target,args);//调用target的method方法，参数为args
        System.out.println("------end--------");
        return result;
    }
    
    //生成代理对象
    public Object getProxy(){
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        Class<?>[] interfaces = target.getClass().getInterfaces();
        return Proxy.newProxyInstance(loader, interfaces, this);
    }
}
```
## 3、使用动态代理  
```
public class ProxyTest{
    public static void main(String[] args){
        Service service = new UserServiceImpl();
        MyInvocationHandler handler = new MyInvocationHandler(service);
        Service serviceProxy = (Service)handler.getProxy();
        serviceProxy.add();
    }
}
```
执行结果：  
```
------before--------
This is add service
------end--------
```
代理对象的生成过程由**Proxy类的newProxyInstance方法**实现，分为三个步骤：  
**①ProxyGenerator.generateProxyClass方法负责生成代理类的字节码**  
```
//proxyName：格式如"com.sun.proxy.$Proxy.1"
//interfaces：代理类需要实现的接口数组
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces);
```
**②native方法Proxy.defineClass0负责字节码加载的实现，并返回对应的Class对象**  
```
Class clazz = defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);
```
**③使用clazz.newInstance反射机制生成代理类对象**  

### 内部实现：  
**1、生成的代理类$Proxy1继承自Proxy类，并且实现了Service接口  
2、执行代理对象的方法serviceProxy.add()，其实就是执行InvocationHandler对象的invoke方法**   
```
public final void add(){
    try{
        super.h.invoke(this, m, null);//传入的参数分别是当前代理对象、当前执行的方法和参数
        return;
    }
    ...
}
```

### InvocationHandler的作用
**在动态代理中，InvocationHandler是核心，每个代理实例都有一个关联的调用处理程序（InvocationHandler）。对代理实例调用方法时，将对方法调用进行编码并将其指派到它的InvocationHandler的invoke()方法。所以对代理方法的调用，内部都是通过InvocationHandler的invoke()实现的，invoke()根据传入的当前代理对象、当前执行的方法和参数来决定调用委托类的哪个方法。**

## jdk动态代理的局限性
通过反射类Proxy和InvocationHandler回调接口实现的jdk动态代理，**要求委托类必须实现一个接口**，但其实并不是每个类都有接口，对于没有实现接口的类，就无法使用这种方式实现动态代理。
