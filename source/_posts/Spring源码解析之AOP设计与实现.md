---
title: Spring源码解析之AOP设计与实现  
date: 2018-07-14 11:20:00  
tags: [Spring,源码解析]    
categories: Spring源码解析
---
### 前言
AOP即面向切面编程，底层通过策略模式选择JDK动态代理还是CGLIB动态代理的方式实现，通过AOP可以在不修改原代码的基础上完成对类的功能的增强。相较于继承这种纵向抽取机制，AOP采用的横向抽取机制更加符合高内聚低耦合的思想。  

### AOP相关概念
- 连接点：类中哪些方法可以被增强，这些方法称为连接点
- 切入点(Pointcut)：类中有很多可以被增强的方法，而实际选用的被增强方法就是切入点
- 通知(Advice)：定义增强的逻辑，即在切入点做什么增强操作，有BeforeAdvice、AfterAdvice、ThrowsAdvice等
- 通知器(Advisor)：将目标方法的切面增强设计(Advice)和切入点(Pointcut)结合起来。通过Advisor，可以定义使用哪个通知并在哪个Pointcut使用它，也就是给指定的Pointcut指定对应的Advice  

<!-- more -->
#### 1、Advice
通知，是AOP的一个基本接口，BeforeAdvice、AfterAdvice、ThrowsAdvice都继承于它。下面以BeforeAdvice为例，在其继承体系中，定义了为待增强目标方法设置的前置增强接口MethodBeforeAdvice。  

使用这个前置接口需要实现一个回调函数**before**()，before方法的实现在Advice中被配置到目标方法后，会在调用木变方法时被回调。  

```
void before(Method method, Object[] args, Object target) throws Throwable;
```
参数：  
Method，是目标方法的反射对象；Object数组，是目标方法的输入参数。  

#### 2、Pointcut
从Pointcut的接口定义中可以知道，需要返回一个**MethodMatcher**，对于Pointcut的匹配判断（即是否需要对当前方法进行增强），都是由这个MethodMatcher来完成的。  

```
public interface Pointcut{
    ClassFilter getClassFilter();
    MethodMatcher getMethodMatcher();
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```
在MethodMatcher接口中，有一个**matches()** 方法，matches方法在匹配连接点的过程中有着至关重要的作用。  

在Pointcut的继承体系中，MethodMatcher可以配置成JdkRegexpMethodPointcut和NameMatchMethodPointcut来完成方法的匹配判断。  

**JdkRegexpMethodPointcut** 通过使用正则表达式来对方法名进行匹配判断的。而**NameMatchMethodPointcut** 通过根据方法的全限定名称来进行匹配判断。  

#### 3、Advisor通知器
以DefaultPointcutAdvisor为例：  

```
public class DefaultPointcutAdvisor ...{
    private Pointcut pointcut = Pointcut.TRUE;
    
    //创建一个匹配所有方法的DefaultPointcutAdvisor
    public DefaultPointcutAdvisor(Advice advice){
        this(Pointcut.TRUE, advice);
    }
    
    //为指定的pointcut和advice创建一个DefaultPointcutAdvisor
    public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice){
        this.pointcut = pointcut;
        setAdvice(advice);
    }
    ...
}
```

### AOP的设计与实现
在使用Spring AOP的过程中，我们可以通过配置达到在目标方法执行前或执行后进行其他操作的目的。这也是AOP完成的流程，首先为目标对象建立代理对象，然后启动代理对象的拦截器来完成各种切面的注入过程。同时，这一系列的织入设计是通过一系列的Adapter来实现的（不同的Advice有不同的Adapter来适配），比如MethodBeforeAdvice就有MethodBeforeAdviceAdapter与之匹配。  
![image](http://osrmzp0jr.bkt.clouddn.com/aopproxy%20generate.jpg)  
在AOP模块中，代理对象的生成主要是通过ProxyFactoryBean来完成的，在其中封装了代理对象的生成过程。在ProxyFactoryBean中，代理对象的生成是以**getObject()** 方法为入口的。  

```
public Object getObject() throws BeansException{
    //在此完成为Proxy对象配置Advisor链
    //这个初始化的过程发生在第一次通过ProxyFactoryBean去获取代理对象的时候
    initializeAdvisorChain();
    
    //对单例和多例类型加以区分，调用不同方法生成代理对象
    if(isSingleton()){
        return getSingletonInstance();
    }else{
        ...
        return newPrototypeInstance();
    }
}
```

如getObject方法中展现的那样，如果是单例对象，那么调用**getSingletonInstance()** 方法生成单例的代理对象，否则调用**newPrototypeInstance()** 生成对象。我们来分析下getSingletonInstance：  

```
private synchronized Object getSingletonInstance(){
    if(this.singletonInstance == null){
        this.targetSource = freshTargetSource();
        ...
        this.singletonInstance = getProxy(createAopProxy());
    }
    return this.singletonInstance;
}

//上面方法通过createAopProxy()返回的AopProxy传入getProxy方法来得到代理对象
protected Object getProxy(AopProxy aopProxy){
    return aopProxy.getProxy(this.proxyClassLoader);
}
```
**关键就在于通过createAopProxy()方法获取到AopProxy**，最终调用的是**DefaultAopProxyFactory**的createAopProxy(config)方法来完成创建AopProxy。  

```
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }
            // 如果 targetClass 是接口类，使用JDK来生成AopProxy
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            // 否则使用CGLIB来生成AopProxy对象
            return new ObjenesisCglibAopProxy(config);
        }
        else {
            return new JdkDynamicAopProxy(config);
        }
    }
```
可以看到在这个方法中，对是否为接口类进行了判断。如果是接口类，就会使用Jdk动态代理生成代理对象；否则，会使用cglib动态代理生成代理对象。通过这个AopProxy对象，将AOP代理对象的生成和框架的其他部门进行了分离。  

这样之后，ProxyFactoryBean的getObject方法返回得到的就不是一个普通的Java对象了，而是一个AopProxy代理对象。这时候就不会让应用只是调用目标方法实现，而是会作为AOP实现的一部分(参照invoke方法的实现)。  

### AOP拦截器链的实现
Spring AOP在通过动态代理生成代理对象的时候，相关的拦截器就已经被配置到代理对象中去了，拦截器在代理对象中起作用是通过对这些方法的回调来完成的。  

#### 拦截器链的获取
在DefaultAdvisorChainFactory中实现了拦截器链的获取过程。首先设置一个List，长度是由配置的通知器的个数决定的，之后通过AdvisorAdapterRegistry来实现拦截器的注册，通过AdvisorAdapterRegistry就行适配，获取到Advisor通知器对应的拦截器，再把它加入到前面的List中去，这样就完成了拦截器的注册。  

#### 拦截器链的调用
如果通过Jdk动态代理的方式生成代理对象，那么需要通过**InvocationHandler的invoke**方法来设置拦截器的回调；如果通过Cglib动态代理的方式生成代理对象，那么就需要通过**DynamicAdvisedInterceptor的intercept**方法来实现拦截器的回调。  

但不管是哪种方式，它们对拦截器链的调用都是在**ReflectiveMethodInvocation**中通过**proceed**方法实现的。在proceed方法中，会逐个运行拦截器的拦截方法。在运行拦截方法之前，需要对代理方法完成一个匹配判断，来决定拦截器是否满足切面增强的要求（通过前面提到的matches方法实现）。在proceed方法中，先进行判断，如果现在运行到拦截器的末尾，那么就会直接调用目标对象的实现方法。  


