---
title: Spring源码解析之IOC容器（一）
date: 2018-04-20 18:45:00  
tags: [Spring,源码解析]    
categories: Spring源码解析
---
## 前言
一直想阅读下Spring的源码，但是翻翻书又感觉这个好复杂，不想去看，所以一直拖到现在，昨天定下心看了下书，觉得并没有那么晦涩难懂，还是要克服惧怕未接触的事物，敢于去克服自己的弱点。主要先看Spring的两大核心IOC和AOP，参考书籍是计文柯的《Spring技术内幕》。这篇文章主要先讲下IOC容器的内部实现。  

## IOC容器简介
IOC即控制反转，这里的反转指的是对象依赖关系的管理被反转了，转到IOC容器上了，对象之间的相互依赖关系由IOC容器进行管理，并由容器完成对象的注入。  

<!-- more -->
## 一、IOC容器的设计与实现
在Spring容器的设计中，有两个主要的容器系列，一个是实现**BeanFactory**接口的简单容器系列，这系列的容器只实现了容器的**最基本功能**；另一个是**ApplicationContext应用上下文**，作为**容器的高级形态而存在**。应用上下文在简单容器的基础上，增加了许多面向框架的特性，同时对应用环境作了许多适配。  

### BeanDefintion
在Spring提供的基本容器的接口定义和实现的基础上，通过定义BeanDefintion来管理Spring应用中的各种对象以及它们之间的相互依赖关系。BeanDefintion抽象了我们对Bean的定义，是让容器起作用的主要数据类型。  

BeanDefintion就是对依赖反转模式中管理的对象依赖关系的数据抽象，是容器实现依赖反转功能的核心数据结构，依赖反转功能都是围绕对这个BeanDefintion的处理来完成的。BeanDefintion就像是容器里装的水，有了这些基本数据，容器才能够发挥作用。

## 二、Spring IOC容器的实现
![image](http://osrmzp0jr.bkt.clouddn.com/ioc1.png)  
- 从接口BeanFactory到HierarchicalBeanFactory到ConfigurableBeanFactory，是一条主要的BeanFactory设计路径。  
    - BeanFactory定义了基本的IOC容器规范，包含了像getBean()这样的IOC容器基本方法。  
    - HierarchicalBeanFactory继承BeanFactory的基本接口之后，增加了getParentBeanFactory()的接口功能，使BeanFactory具备了双亲IOC容器的管理功能。
    - ConfigurableBeanFactory接口中，主要定义一些对BeanFactory的配置功能，如setParentBeanFactory()设置双亲IOC容器，通过addBeanPostProcessor()配置Bean后处理器等。  

- 第二条设计主线是，以ApplicationContext应用上下文接口为核心的接口设计，涉及的主要接口设计有，从BeanFactory到ListableBeanFactory，再到ApplicationContext，再到我们常用的应用上下文WebApplicationContext和ConfigurableApplicationContext。
    - 在ListableBeanFactory接口中，细化了许多BeanFactory的接口功能，比如getBeanDafiinitions()。
    - 对于ApplicationContext接口，通过继承MessageSource、ResourceLoader、ApplicationEventPublisher接口，在BeanFactory简单容器基础上添加了许多对高级容器特性的支持。

- 在ApplicationContext的设计中，一方面，可以看到它集成了BeanFactory接口体系中的ListableBeanFactory、HierarchicalBeanFactory、AutowireCapableBeanFactory等BeanFactory的接口，具备了BeanFactory IOC容器的基本功能；另一方面，通过继承MessageSource、ResourceLoader、ApplicationEventPublisher接口，具备更高级的特性。  

### ①BeanFactory
![image](http://osrmzp0jr.bkt.clouddn.com/beanF.png)  
上图中定义的接口方法勾画出了IOC容器的基本特性。
#### BeanFactory的设计原理
以XmlBeanFactory为例，说明简单IOC容器的设计原理。  

![image](http://osrmzp0jr.bkt.clouddn.com/xmlbf.png)  
XmlBeanFactory继承自**DefaultListableBeanFactory**这个类，DefaultListableBeanFactory十分重要，是经常要用到的一个IOC容器的实现，包含了基本IOC容器所具有的重要功能。在Spring中，DefaultListableBeanFactory作为一个默认的功能完整的IOC容器来使用，在其他容器，如ApplicationContext中，也是通过持有或扩展DefaultListableBeanFactory来获得基本的IOC容器的功能的。  

XmlBeanFactory是一个可以读取以XML文件方式定义的BeanDefinition的IOC容器。XML读取功能是如何实现的呢？  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%883.56.21.png)  
在XmlBeanFactory中，初始化一个XmlBeanDefinitionReader对象用于处理XML，在构造XmlBeanFactory时，指定BeanDefinition的信息来源，这个信息来源需要封装成Resource类来给出，然后将Resource作为构造参数传递给XmlBeanFactory构造函数，这样IOC容器就可以方便的定位到需要的BeanDefinition信息，来对Bean完成容器的初始化和依赖注入的过程。  

### ②ApplicationContext  
ApplicationContext在BeanFactory的基础上添加了附加功能，使其成为高级形态意义的容器。提供了以下BeanFactory不具备的特性：  
- 支持不同的信息源。ApplicationContext扩展了MessageSource接口，使其可以支持国际化的实现，为开发多语言版本的应用提供服务。
- 访问资源。体现在对ResourceLoader和Resource的支持上，这样我们可以从不同地方得到Bean定义资源。
- 支持应用事件。继承接口ApplicationEventPublisher，在上下文中引入了事件机制。与Bean生命周期的结合为Bean的管理提供便利。
- 在ApplicationContext中提供的附加服务。

#### ApplicationContext设计原理
以FileSystemXmlApplicationContext的实现为例来说明设计原理。主要基本功能已经在其基类AbstractXmlApplicationContext中实现了，作为一个具体的应用上下文，只需要实现和它自身设计相关的两个功能。  
- 一个功能是：如果直接使用FileSystemXmlApplicationContext，对于实例化这个应用上下文的支持，同时会启动IOC容器的refresh()过程，这个refresh()过程会涉及到IOC容器启动的一系列复杂操作。
- 另一个功能是：是FileSystemXmlApplicationContext设计具体相关的功能，这部分定义如何从文件系统中加载XML的Bean定义资源。

通过这个过程，可以为在文件系统中杜徐以XML形式存在的BeanDefinition做准备，由于不同的应用上下文实现对应着不同的读取BeanDefinition的方式。FileSystemXmlApplicationContext的读取方式如下：  
```
protected Resource getResourceByPath(String path){
    if(path != null && path.startsWith("/")){
        path = path.substring(1);
    }
    return new FileSystemResource(path);
}
```
这样，就可以得到FileSystemResource的资源定位了。

## 二、IOC容器的初始化过程
IOC容器的初始化是通过refresh()方法来启动的，这个启动包括BeanDefinition的Resource定位、载入和注入三个基本过程。  
1. 第一个过程：Resource的定位过程。指的是BeanDefinition的资源定位，由ResourceLoader通过统一的Resource接口来完成，这个Resource对各种形式的BeanDefinition的使用提供了统一接口。  
2. 第二个过程：BeanDefinition的载入。这个载入过程是把定义好的Bean表示成IOC容器内部的数据结构，而这个容器的内部数据结构就是BeanDefinition。
3. 第三个过程：向IOC容器注册这些BeanDefinition的过程。这个过程通过BeanDefinitionRegistry接口的实现来完成。在IOC容器内部把BeanDefinition注入到一个HashMap中去，IOC容器就是通过这个HahshMap来持有这些BeanDefinition数据的。  

这里谈的IOC容器初始化的过程，一般不包含Bean的依赖注入的实现，**依赖注入一般发生在应用第一次通过getBean向容器索取Bean的时候**。除非设置了lazyinit属性，那么这个Bean的依赖注入在IOC容器初始化时就预先完成了。  

### 1、BeanDefinition的Resource定位
以编程的方式实现DefaultListableBeanFactory时，首先要定义一个Resource来定位容器使用的BeanDefinition，这时使用的是ClassPathResource，意味着Spring会在类路径中去寻找以文件形式存在的BeanDefinition信息。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%883.56.31.png)  

在ApplicationContext中，Spring已经提供好了一系列加载不同Resource的读取器的实现，不需要为它再配置特定的读取器才能完成功能。

以FileSystemApplicationContext为例，看看是如何实现Resource定位的：  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%884.47.21.png)  

refreshBeanFactory()在refresh中被调用，在该方法中，首先通过createBeanFactory构建一个IOC容器供ApplicationContext使用，即前面提到的DefaultListableBeanFactory，同时启动loadBeanDefinitions来载入BeanDefinition。  
### 2、BeanDefinition的载入与解析
在完成对BeanDefinition的Resource定位的分析后，进行整个BeanDefinition信息的载入过程，这个载入过程，相当于将定义好的BeanDefinition在IOC容器中转化为一个Spring内部表示的数据结构的过程。refresh()方法是IOC容器初始化的开始：  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%885.13.45.png)  

载入过程的开始，可以从FileSystemApplicationContext的基类AbstractRefreshableApplicationContext的loadBeanDefinitons()方法中看到，可以从这个基类的refreshBeanFactory()开始，了解这个Bean定义信息载入的过程。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%885.22.02.png)  

在读取器reader中，得到代表XML文件的Resource，读取器在打开IO流后得到XML的文件对象。有了这个文件对象后，就可以按照Spring的Bean定义规则来对这个XML文档树进行解析了，这个**解析是交给BeanDefinitionParserDelegate的registerBeanDefinitions来完成的**。  

使用DefaultDocumentLoader得到Document对象，**那么Spring是怎样按照Bean的语义进行解析并转化为容器内部数据结构的？** 这个过程在registerBeanDefinitions(doc, resource)中完成的。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%885.33.10.png)  
载入过程分为两部分：首先通过调用XML解析器得到document对象；完成XML解析后，按照Spring的Bean规则进行解析，这个解析是在documentReader.registerBeanDefinitions()中具体实现的（见上面代码）。  

解析处理完成后，解析结果会放到BeanDefinition对象中，并由BeanDefinitionHolder对象来持有，这个BeanDefinitionHolder除了持有BeanDefinition对象外，还持有其他相关信息，如Bean的名字、别名集合等。  

通过逐层的解析，我们在XML文件中定义的BeanDefinition就被整个载入到了IOC容器中，并在容器中建立数据映射。通过以上的载入过程，IOC容器大致完成了管理Bean对象的数据准备工作。

### 3、BeanDefinition在IOC容器中的注册
经过BeanDefinition在IOC容器的载入和解析，用户定义的BeanDefinition信息已经在IOC容器里建立起了自己的数据结构以及对应的数据表示，但此时这些数据还不能供IOC容器直接使用，需要在IOC容器进行注册。在DefaultListableBeanFactory中，是通过一个HashMap来持有载入的BeanDefinition的。

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%886.02.19.png)  

## 三、小结
今天主要看完了IOC容器系列的设计和实现，包括BeanFactory和ApplicationContext；以及IOC容器的初始化过程，包括BeanDefinition的Resource定位，BeanDefinition的载入与解析，还有BeanDefinition的注册。  

其中BeanDefinition比较抽象，它是对象依赖关系的一种数据抽象，通过定义BeanDefinition来管理Spring应用中各种对象以及它们之间的相互依赖关系。  

IOC容器初始化的过程，下面这段代码是一个基本思路。通过Resource读取器读取Resource，进行载入。得到Resource后，按照Spring的Bean定义规则来对这个XML文档树进行解析，解析是交给BeanDefinitionParserDelegate类下的registerBeanDefinitions(doc, resource)，内部又调用了documentReader.registerBeanDefinitions(doc, resource)（内部具体实现又为processBeanDefinition()）来完成（见2-14代码），最后通过registerBeanDefinition（无s，与上面不同）来进行BeanDefinition的注册。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%883.56.31.png)   

![image](http://osrmzp0jr.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-20%20%E4%B8%8B%E5%8D%885.33.10.png)  
