---
title: Java反射机制  
date: 2017-08-24 11:26:04  
tags: [java]    
categories: java  
---
# 一、反射技术简介及获取Class对象
**反射技术是指动态的获取指定的类，以及动态的调用类中的内容。**  
要想获取字节码文件的成员，就必须先获取到字节码文件对象（Class对象）。  
**Class对象：虚拟机在class文件的加载阶段，把类信息保存在方法区的数据结构中，并在Java堆中生成一个Class对象，作为类信息的入口**。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%8F%8D%E5%B0%84.png)  
介绍三种获取字节码文件对象的方式：  
首先声明一个Person类  

```
class Person{
    //字段
    private String name;
    private int age;
    
    //构造函数
    Person(){ ... }
    Person(String name, int age){ ... }
    
    //方法
    public void show(String name, int age){ ... }
    public static void staticShow(){ ... }
}
```

## 1、通过实例变量的方式
```
Person person = new Person();
Class clazz = person.getClass();
```
使用Object类中的getClass方法，是通用的，但前提是必须有指定类，并对该类进行对象的创建，才能调用getClass方法。  

## 2、通过类名的方式
```
Class clazz = Person.class
```
不用new对象，但还是需要使用具体的类。这种方式，只会加载Person类，但不会触发它的类构造器来初始化。    

## 3、通过Class类中的forName(String className)的方式  
```
String classNmae = "cn.wunian.domian.Person";//要用全限定名
Class clazz = Class.forName(className);
//通过newInstance()方法就可以创建字节码文件对象所表示的类的实例
Object obj = clazz.newInstance();//创建字节码文件对象所表示的类的一个实例
```
**这种方式是反射技术所使用的获取字节码文件对象的方式。只需要知道类的名字就可以了，获取对应的字节码文件等由forName方法自动完成。**  

总结下，主要分为三个步骤：  
**①通过给定的类名称，加载对应的字节码文件，并封装成字节码文件对象Class  
②通过newInstance创建字节码文件对象所表示的类的实例  
③调用该类的构造函数来进行初始化**  

# 二、反射机制
反射机制可以在**运行期间获取类的字段、方法、父类和接口**等信息。  
## 1、获取构造函数
通常被反射的类都会提供空参数的构造函数。如果没有对应的构造函数，会报InstantiationException；如果有提供，但访问权限不够，会报IllegalAccessException。  
```
//使用带参的构造器来初始化对象
String className = "cn.wunian.domain.Person";
Class clazz = Class.forName(className);
Constructer constructer = clazz.getConstructer(String.calss, int.class);
//通过构造器对象来初始化对象
Object obj = constructer.newInstance("xubo", 24);
```
使用此Constructer对象表示的构造方法来创建该构造方法的声明类的新实例，并用指定的初始化参数来初始化该实例。  

## 2、获取字段
```
String className = "cn.wunian.domain.Person";
Class clazz = Class.forName(className);
String fieldName = "age";
Object obj = clazz.newInstance();
//获取字段对象
Field field = clazz.getDeclaredField(fieldName);
//取消访问检查
field.setAccessible(true);
//设置字段值
field.set(obj, 30);//将指定对象变量上此Field对象表示的字段设置为指定的新值
//返回指定对象上此Field字段的值
System.out.println(field.get(obj));
```

## 3、获取方法
用到的两个方法：  
①返回一个Method对象，它反映此Class对象所表示的类或接口指定的名为methodName的公共成员方法  
```
Class.getMethod(String methodName, class<?>...parameterTypes)
methodName：方法名  
parameterTypes：参数列表所属的类型
```
②对带有指定参数（parameterTypes）的指定对象（obj）调用由此Method对象表示的底层方法（上面的methodName代表的方法）  
```
Method.invoke(Object obj, Object...args)
obj：调用底层方法的对象  
args：用于方法调用的参数
```
### ①获取带有参数的方法
```
String className = "cn.wunian.domain.Person";
Class clazz = Class.forName(className);
Object obj = clazz.newInstance();
String methodName = "show";
Method method = clazz.getMethod(methodName, String.class, int.class);
method.invoke(obj, "xubo", 24);
```
### ②获取无参数的方法
```
String className = "cn.wunian.domain.Person";
Class clazz = Class.forName(className);
String methodName = "staticShow";
Method method = clazz.getMethod(methodName, null);//无参传null
method.invoke(null,null)//静态方法不需要对象调用，传null
```

# 三、反射的性能问题
1、代码的语言访问检查过于复杂，本来这块应该在链接阶段实现的，使用反射时需要在运行时才进行  
2、运行时进行验证会产生过多的临时对象，影响GC的消耗  
3、由于缺少上下文，导致不能进行更多的优化  
现代JVM已经不慢了，能对反射代码进行缓存以及通过方法计数器同样实现JIT优化，所以反射不一定慢，可以大胆使用反射。








