---
title: cglib动态代理  
date: 2017-08-25 17:22:04  
tags: [java，java基础]    
categories: java  
---
# 一、cglib实现动态代理步骤
使用cglib实现动态代理，并不要求委托类必须实现接口，底层使用asm字节码生成框架生成代理类的字节码。  
## 1、定义业务逻辑
```
public class UserService{
    public void add(){
        System.ou.println("This is add service");
    }
}
```
## 2、实现MethodInterceptor接口，定义方法的拦截器  
```
public class MyMethodInterceptor implements MethodInterceptor{
    public Object intercept(Object obj, Method method, Object[] arg, MethodProxy proxy)throws Throwable{
        System.ou.println("Before:" + method);
        Object object = proxy.invokeSuper(obj, arg);
        System.ou.println("After:" + method);
        return object;
    }
}
```
## 3、使用Enhancer类生成代理类
```
Enhancer enhancer = new Enhancer();
enhancer.setSuperClass(UserService.class);
enhancer.setCallback(new MyMethodInterceptor());
UserService userService = (UserService)enhancer.create();
```
## 4、userService.add()的执行结果
```
Before: add
This is add service
After: add
```

# 二、cglib字节码生成
**Enhancer**是cglib字节码增强器，可以方便的对类进行扩展，**内部调用GeneratorStrategy.generate方法生成代理类的字节码**  
内部实现：  
**1、代理类UserService$$EnhancerByCGLIB$$394dddeb继承了委托类UserService，且委托类的final方法不能被代理；  
2、代理类为每个委托方法都生成两个方法，以add方法为例，一个是重写的add方法，一个是CGLIB$add$0方法，该方法直接调用委托类的add方法；  
3、当执行代理对象的add方法时，会先判断是否存在实现了MethodInterceptor接口的对象cglib$CALLBACK_0（也就是这里的MyMethodInterceptor），如果存在，那么调用MethodInterceptor对象的intercept方法**
```
public Object intercept(Object obj, Method method, Object[] arg, MethodProxy proxy)throws Throwable{
    System.ou.println("Before:" + method);
    Object object = proxy.invokeSuper(obj, arg);//最终调用的委托类中的method方法
    System.ou.println("After:" + method);
    return object;
}
```
参数分别是：①代理对象；②委托类方法；③方法参数；④代理方法的MethodProxy对象  
**4、每个被代理的方法都对应一个MethodProxy对象，methodProxy.invokeSuper最终调用的是委托类的add方法**，实现如下：  
```
public Object invokeSuper(Object obj, Object[] args)throws Throwable{
    try{
        init();
        FastClassInfo fci = fastClassInfo;
        //fci.f2:代理类对象
        //fci.i2:方法CGLIB$add$0在对象中的索引位置
        return fci.f2.invoke(fci.i2, obj, args);//调用代理类对象的CGLIB$add$0方法，CGLIB$add$0方法又直接调用委托类的add方法
    }
    ...
}
```
在MethodProxy实现中，通过FastClassInfo维护了委托类和代理类的FastClass：  
```
private static class FastClassInfo{
    FastClass f1;//指向委托类对象
    FastClass f2;//指向代理类对象
    int i1;//方法add在对象中的索引位置
    int i2;//方法CGLIB$add$0在对象中的索引位置
}
```
**FastClass提出概念下标index，通过索引来保存方法的引用信息，将原先的反射调用，转化为方法的直接调用，从而实现fast。当调用methodProxy.invokeSuper方法时，实际上是调用代理类的CGLIB$add$0方法，CGLIB$add$0方法又直接调用委托类的add方法，避免了使用反射机制来调用委托类的方法。**  

# 三、jdk和cglib动态代理的区别
1、jdk动态代理生成的代理类和委托类必须实现相同的接口；cglib动态代理则不需要。  
2、cglib动态代理中，生成的代理类是委托类的子类，且不能处理被final关键字修饰的方法；  
3、jdk采用反射机制来调用委托类的方法，cglib采用类似索引的方式直接调用委托类的方法。




