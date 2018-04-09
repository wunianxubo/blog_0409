---
title: Java中的异常和处理  
date: 2017-10-09 16:49:00  
tags: [java,java基础]    
categories: java基础  
---
# 简介
程序运行时，发生超出预期的事件，阻止了程序按照程序员的预期正常执行，这就是异常。Java中的解决方法为**异常处理机制**。  
异常处理机制能让程序在异常发生时，按照代码预先设定的异常处理逻辑，针对性的处理异常，让程序尽最大可能恢复正常并继续执行，且保持代码的清晰。  
java中的异常可以是函数中的语句执行时引发的，也可以是程序员通过throw关键字手动抛出的，只要程序中产生了异常，就会用一个对应类型的异常对象来封装异常，jre就会试图寻找异常处理程序来处理异常。  
Throwable类是Java异常类型的顶层父类，一个对象只有是Throwable类的实例，他才是一个异常对象，才能被异常处理机制识别。jdk中内置了一些常用的异常类，同时程序员自己也可以自定义异常。  
# java异常的分类和类结构图
Throwable类是顶层父类，Throwable类派生出Error类和Exception类。  
**错误**：Error类以及他的子类的实例，代表了JVM本身的错误。错误不能被程序员通过代码处理，Error很少出现。  
**异常**：Exception以及它的子类，代表程序运行时发送的各种不期望发生的事件。**可以被java异常处理机制使用，是异常处理的核心**。  
![image](http://osrmzp0jr.bkt.clouddn.com/%E5%BC%82%E5%B8%B8%E4%BD%93%E7%B3%BB.jpg)  
根据javac对异常的处理要求，将异常类分为2类：  
**非检查异常（unchecked exception）**：Error和RuntimeException以及他们的子类。javac在编译时，不会提示和发现这样的异常，不要求在程序中处理这些异常。对于这样的异常，我们可以编写代码处理（try...catch...finally），也可以不处理。对于这些异常，我们要做的应该是修正代码，而不是去通过异常处理器处理。这样的异常多半是代码编写的问题。如除0错误ArithmeticException，错误的类型强转错误ClassCastException，数组越界错误ArrayIndexOutOfBoundsException，空指针错误NullPointerException等等。  
**检查异常（checked exception）**：除了Error和RuntimeException的其它异常。javac强制要求程序员为这样的异常做预备处理工作（try...catch...finally）。在方法中，**要么通过try-catch语句捕获异常并处理，要么用throws子句声明交给函数调用者去解决**，否则编译不会通过。这种异常一般由程序的运行环境导致的。因为程序可能被运行在各种未知的环境下，但是程序员无法干预用户如何使用他编写的程序，于是程序员就应该为这样的异常时刻准备着，如SQLException，IOException，ClassNotFoundException，FileNotFoundException等等。  
这里的检查与非检查是针对javac而言的。
