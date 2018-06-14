---
title: Spring源码解析之IOC容器（二）
date: 2018-05-11 11:53:00  
tags: [Spring,源码解析]    
categories: Spring源码解析
---
## 前言
上一篇，我们主要分析了IOC容器初始化过程中，BeanDifinitions的定位、载入解析以及注册的流程。在这篇文章，我们会来谈一谈IOC容器的依赖注入及其他相关特性的分析。

## IOC容器的依赖注入
依赖注入的过程是用户第一次向IOC容器索要Bean时出发的，即使用getBean()方法时。当然也有例外，当我们设置了lazy-init属性后，会让容器完成对Bean的预实例化，这个这个预实例化其实也是一个依赖注入的过程，只是时机和场景不同。（可见refresh方法中的注解）  
<!-- more -->

```
pulic void refresh() throws BeansException{
    ...
    prepareRefresh();
    
    //这里是子类中启动refreshBeanFactory()的地方，这个方法主要负责：
    //①BeanFactory如何已存在就更新，不存在就创建。
    //②其中有个loadBeanDefinitions(beanFactory)方法，在第一篇中提到了该方法，用来加载解析Bean的定义，把用户自定义的数据结构转换成容器中特定的数据结构。
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    
    //此方法用来添加一些Spring本身需要的工具类
    prepareBeanFactory(beanFactory);
    
    //下面两个方法用于对已经构建的BeanFactory的配置做修改
    //设置BeanFactory的后置处理
    postProcessBeanFactory(beanFactory);
    //调用BeanFactory的后处理器，这些后处理器是在Bean定义中向容器注册的
    invokeBeanFactoryPostProcessors(beanFactory);
    
    //注册Bean的后处理器，在Bean的创建过程中调用。
    //获取用户定义的实现了BeanPostProcessors接口的子类，接口中实现了两个方法postProcessBeforeInitialization和
    //postProcessAfterInitialization分别在Bean对象初始化时和初始化后执行，在此可自己定义操作。（可结合下面Bean的生命周期一起理解）
    registerBeanPostProcessors(beanFactory);
    
    //对上下文中的消息源进行初始化
    initMessageSource();
    
    //初始化上下文中的事件机制
    initApplicationEventMulticaster();
    
    //初始化其他特殊Bean
    onRefresh();
    
    //检查监听Bean并且将这些Bean向容器注册
    registerListeners();
    
    //实例化所有的（non-lazy-init）单例，一般情况下不会进行依赖注入；如果设置了lazy-init属性，会直接在这使用getBean()去触发依赖注入，
    //而一般依赖注入是发生在IOC容器初始化后，第一次执行getBean时，两者时机不同，场景不同。
    //在这里只是完成Bean生命周期的第一步——实例化
    finishBeanFactoryInitialization(beanFactory);
    
    //发布容器事件，结束refresh过程
    finishRefresh;
    
    ...
}
```

### 一、Bean对象关系依赖注入的流程
![image](http://osrmzp0jr.bkt.clouddn.com/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E6%B5%81%E7%A8%8B.png)


### 二、Spring bean的生命周期
![image](http://osrmzp0jr.bkt.clouddn.com/spring.jpg)   
首先，Bean的生命周期可以简单分为以下几步：  
- Bean实例的创建（对应下面的1）  
- 为Bean实例设置属性（对应下面的2~5）  
- 调用Bean的初始化方法（对应下面的6~8）  
- 应用通过Ioc容器使用Bean（对应下面的9）  
- 当容器关闭时，调用Bean的销毁方法（对应下面的10）    

#### 具体步骤如下
1. Spring对Bean进行实例化（相当于程序中的new Xx()）  
2. Spring将值和Bean的引用注入进Bean对应的属性中（polulateBean()对对象依赖关系进行注入）  
//3~5 Spring会检测该对象是否实现了xxxAware接口，并将相关的xxxAware实例注入给bean
3. 如果Bean实现了BeanNameAware接口，Spring将Bean的ID传递给setBeanName()方法（实现BeanNameAware清主要是为了通过Bean的引用来获得Bean的ID，一般业务中是很少有用到Bean的ID的）  
4. 如果Bean实现了BeanFactoryAware接口，Spring将调用setBeanDactory(BeanFactory bf)方法并把BeanFactory容器实例作为参数传入。（实现BeanFactoryAware 主要目的是为了获取Spring容器，如Bean通过Spring容器发布事件等）  
5. 如果Bean实现了ApplicationContextAware接口，Spring容器将调用setApplicationContext(ApplicationContext ctx)方法，把y应用上下文作为参数传入.(作用与BeanFactory类似都是为了获取Spring容器，不同的是Spring容器在调用setApplicationContext方法时会把它自己作为setApplicationContext 的参数传入，而Spring容器在调用setBeanDactory前需要程序员自己指定（注入）setBeanDactory里的参数BeanFactory )  
6. 如果Bean实现了BeanPostProcess接口，Spring将调用它们的postProcessBeforeInitialization（预初始化）方法（作用是在Bean实例创建成功后对进行增强处理，如对Bean进行修改，增加某个功能）  
7. 如果Bean实现了InitializingBean接口，Spring将调用它们的afterPropertiesSet方法，作用与在配置文件中对Bean使用init-method声明初始化的作用一样，都是在Bean的全部属性设置成功后执行的初始化方法。  
8. 如果Bean实现了BeanPostProcess接口，Spring将调用它们的postProcessAfterInitialization（后初始化）方法（作用与6的一样，只不过6是在Bean初始化前执行的，而这个是在Bean初始化后执行的，时机不同 )  
9. 经过以上的工作后，Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁  
10. 如果Bean实现了DispostbleBean接口，Spring将调用它的destory方法，作用与在配置文件中对Bean使用destory-method属性的作用一样，都是在Bean实例销毁前执行的方法。  

### 三、FactoryBean
FactoryBean是特殊的Bean，是一个工厂Bean，可以产生Bean的Bean。如果一个类继承FactoryBean，用户可以自己定义产生实例对象的方法，只需实现这个类的getObject方法。通过调用这个对象的getObject方法就能获取用户自定义产生的对象，为Spring提供了很好的扩展性。  
#### FactoryBean和BeanFactory的区别？
BeanFactory是Ioc容器最基本的实现，负责生产和管理bean，为其他具体的Ioc容器提供了最基本的规范，例如DefaultListableBeanFactory,XmlBeanFactory,ApplicationContext 等具体的容器都是实现了BeanFactory，再在其基础之上附加了其他的功能。  

FactoryBean是一个接口，当在IOC容器中的Bean实现了FactoryBean后，通过getBean(String BeanName)获取到的Bean对象并不是FactoryBean的实现类对象，而是这个实现类中的getObject()方法返回的对象。要想获取FactoryBean的实现类，就要getBean(&BeanName)，在BeanName之前加上&。FactoryBean在IOC容器的基础上给Bean的实现加上了一个简单工厂模式和装饰模式，我们可以在getObject()方法中灵活配置。  

FactoryBean（通常情况下，Bean无须自己实现工厂模式，Spring容器担任工厂角色；但少数情况下容器的Bean本身就是工厂，其左右是产生其他Bean实例），这种情况下，bean不需要特别的要求，只需提供一个工厂方法，该方法用来返回其他Bean实例，由工厂bean产生的bean实例，不再由Spring容器产生，所以不用在配置文件中提供class元素。   

### 四、BeanPostProcessor
这是Bean的后置处理器，是一个监听器，可以监听容器触发的事件。将它向IOC容器注册后，容器中管理的Bean具备了接收IOC容器事件回调的能力。BeanPostProcessor是一个接口类，有两个接口方法，分别是postProcessBeforeInitialization和postProcessAfterInitialization，分别在Bean的初始化前和初始化后提供回调入口。postProcessBeforeInitialization和populateBean之间的关系如下：  

```
Object exposedObject = bean;  
populateBean(beanName, mbd, instanceWrapper);  
//在完成对Bean的生成和依赖注入时候，开始对Bean进行初始化，这个初始化过程包括了对
//postProcessBeforeInitialization的回调。即Bean生命周期中before - initialize - after这种顺序
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

### 五、autowiring（自动依赖装配）
在自动装配中，不需要对Bean属性做显式的依赖关系声明，只需要配置好autowiring属性，IOC容器会根据这个属性的配置，使用反射自动查找属性的类型或者名字，然后基于属性的类型或者名字来自动匹配IOC容器中的Bean，从而自动的完成依赖注入。  

在populateBean的实现中，在处理一般的Bean之前，先对autowiring属性进行处理。如果当前的Bean配置了autowire_by_name和autowire_by_type，会相应的调用autowireByName或者autowireByType方法。  

以autowireByName为例，进行以下几个步骤：  
- 首先得到Bean的属性名，这些信息在BeanWrapper和BeanDefinition中已经封装好了
- 使用属性名作为Bean名字使用getBean()方法向容器索取Bean，这个操作会触发当前Bean依赖的Bean的依赖注入，从而得到属性对应的依赖Bean
- 最后，把这个依赖Bean注入到当前Bean的属性中去。完成操作！