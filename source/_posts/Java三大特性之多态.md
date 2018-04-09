---
title: Java三大特性之多态性  
date: 2017-12-01 20:38:00  
tags: [java,java基础]    
categories: java基础  
---
# 一、多态的定义
&nbsp;&nbsp;多态就是指：程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定，即一个引用变量到底会指向哪个类的实例对象，该引用变量发出的方法到底是哪个类中实现的方法，必须由程序运行期间才能决定。这样，不需要修改源程序，就可以让引用变量绑定到各种不同的类实现上，从而导致该引用调用的具体方法随之改变，即不修改程序代码就可以改变程序运行时所绑定的具体代码，让程序选择多个运行状态，这就是多态性。  
# 二、多态的实现条件
Java实现多态有三个必要条件：**继承、重写、向上转型**。  
继承：在多台中必须存在有继承关系的子类和父类。  
重写：子类对父类中的某些方法进行重新定义，在调用这些方法时就会调用子类的方法。  
向上转型：在多态中需要把父类型的引用变量指向子类对象。  
&nbsp;&nbsp;多态的实现机制遵循一个原则：当父类对象的引用变量指向子类对象时，子类对象的类型决定了调用的是谁的方法，但是这个被调用的方法必须是在父类中定义过的，也就是说被子类覆盖的方法。  
# 三、经典实例
&nbsp;&nbsp;**原则：当父类对象的引用变量指向子类对象时，子类对象的类型决定了调用的是谁的方法，但是这个被调用的方法必须是在父类中定义过的，也就是说被子类覆盖的方法。如果不符合前面的条件，它仍然需要根据继承链中方法调用的优先级来确认方法，优先级为：**
```
//这类this指的是引用变量的类型
this.show(o)--->super.show(o)--->this.show((super)o)--->super.show((super)o)
```

```
public class A {
    public String show(D obj) {
        return ("A and D");
    }

    public String show(A obj) {
        return ("A and A");
    } 

}

public class B extends A{
    public String show(B obj){
        return ("B and B");
    }
    
    public String show(A obj){
        return ("B and A");
    } 
}

public class C extends B{

}

public class D extends B{

}

public class Test {
    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new B();
        B b = new B();
        C c = new C();
        D d = new D();
        
        System.out.println("1--" + a1.show(b));
        System.out.println("2--" + a1.show(c));
        System.out.println("3--" + a1.show(d));
        System.out.println("4--" + a2.show(b));
        System.out.println("5--" + a2.show(c));
        System.out.println("6--" + a2.show(d));
        System.out.println("7--" + b.show(b));
        System.out.println("8--" + b.show(c));
        System.out.println("9--" + b.show(d));      
    }
}

//运行结果
1--A and A
2--A and A
3--A and D
4--B and A
5--B and A
6--A and D
7--B and B
8--B and B
9--A and D
```
&nbsp;&nbsp;比如分析4，a2.show(b)，按照上面原则的意思，是由B来决定调用谁的方法，所有a2.show(b)应该要调用B中的show(B obj)，产生的结果应该是“B and B”，但是这样是错误的，因为我们忽略了后面那句话：被调用的方法必须要在父类中定义过的，不符合要求，不能直接这么做！所以仍然要按照继承链的调用方法的优先级来确认。由于this(A)没有父类，跳到第三级this((super)o)，找到了A中的show(A obj)方法，同时由于B重写了该方法，所以最终会调用B类中的show(A obj)方法，最后输出“B and A”。
