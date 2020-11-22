---
title: HOOK初探
date: 2020-10-02 17:57:49
tags:
	- Hook
	- 设计模式
categories: 学习
---

### 引入

#### hook methods？

在翻阅文档的时候，我注意到了一个直译意译都没法理解的词：

<img src="https://s1.ax1x.com/2020/10/02/0lwfNn.png" alt="0lwfNn.png" style="zoom:33%;" />

>一个帮助实现AppWidget提供程序的便利类。可以使用AppWidgetProvider进行的所有操作，也是使用常规的BroadcastReceiver可以进行的操作。AppWidgetProvider仅从onReceive(Context, Intent)中接收到的Intent中解析出相关字段，然后使用接收到的附加数据调用**hook（钩子）方法**。
>
>扩展此类并重写onUpdate(Context, AppWidgetManager, int [])，onDeleted(Context, int [])，onEnabled(Context)或onDisabled(Context)方法中的一个或多个，以实现自己的AppWidget功能。

<!--more-->

调用hook methods？这也没有解释什么是hook methods，那我们来Google一下：

<img src="https://s1.ax1x.com/2020/10/02/0l0HRP.png" alt="0l0HRP.png" style="zoom:33%;" />

>**Hooks methods customise generic framework components to run application-specific logic.** Example of a hook method is the run() method inside a thread in java. It's up to you to add application-specific logic inside this method.
>
>**钩子方法定制通用框架组件来运行特定于应用程序的逻辑。**钩子方法的例子是java线程中的run()方法。在此方法中添加特定于应用程序的逻辑由您决定。

<img src="https://s1.ax1x.com/2020/10/02/0l07Gt.png" alt="0l07Gt.png" style="zoom:33%;" />

> What is a Hook Method? **Hook methods provide a way to extend behavior of programs at runtime.** Imagine having the ability to get notified whenever a child class inherits from some particular parent class or handling non-callable methods on objects elegantly without allowing the compiler to raise exceptions.
>
> 什么是钩子方法？**钩子方法提供了一种在运行时扩展程序行为的方法。**想象一下，只要子类继承了某些特定的父类，或者优雅地处理对象上的不可调用方法，而不允许编译器引发异常，就可以获得通知。

专业概念一般是不会让人看懂是什么意思的，但是可以看到几个关键词：**通用**、**特定**、**运行时**、**扩展**，初步感觉像是接口、抽象方法这种提供模版来个性化实现相同功能的，但肯定是有很大区别的。

### Hook机制

我们先来了解一下什么是Hook：[一个讲的比较清楚的博客](https://www.rswebsols.com/tutorials/programming/software-development-hook-hooking#lwptoc1)

> **什么是挂钩？** 在软件开发中，挂钩是一个允许修改程序行为的概念。代码有机会使您更改某些东西的原始行为，而无需更改相应类的代码。这是通过覆盖hook方法来完成的。
>
> 在向应用程序添加新功能的情况下，这种类型的实现非常有用，它还可以促进其他进程与系统消息之间的通信。挂钩通常会通过增加系统需要为每个消息执行的处理负载来降低系统性能。仅应在需要时安装它，并尽快将其卸下。
>

#### Example

```java
public class Algorithm {
  public void templateMethod() {
    hookMethod();
  }
  
  public void hookMethod() {
    // default implementation
  }
}


public class RefinedAlgorithm extends Algorithm {
  public void hookMethod() {
    // refined implementation
  }
}
```

