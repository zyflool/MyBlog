---
title: Context相关
date: 2020-04-04 15:20:27
tags: 
	- Android
	- Context
Categories: 学习
---

#### 概念

> Interface to global information about an application environment.  This is an abstract class whose implementation is provided by the Android system.  It allows access to application-specific resources and classes, as well as up-calls for application-level operations such as launching activities,  broadcasting and receiving intents, etc.
>
> 与有关应用程序环境的全局信息的接口。 这是一个抽象类，其实现由Android系统提供。 它允许访问特定于应用程序的资源和类，以及对应用程序级操作（例如启动活动，广播和接收意图等）的调用。

<!--more-->

![context关联类](https://i.loli.net/2020/04/04/umbThWireoCa9qA.jpg)

ContextWrapper类中的mBase具体指向ContextImpl。ContextImpl提供了很多功能，但是外界需要使用并拓展ContextImpl的功能，因此设计上使用了装饰模式，ContextWrapper是装饰类，它对ContextImpl进行包装，ContextWrapper主要是起了方法传递的作用，ContextWrapper中几乎所有的方法都是调用ContextImpl的相应方法来实现的。

Context的关联类使用了装饰模式，有以下的优点：

+ 使用者（如Service）能更方便地使用Context。
+ 如果ContextImpl发生了变化，它的装饰类ContextWrapper不需要做任何修改。
+ ContextImpl的实现不会暴露给使用者，使用者也不必关心ContextImpl的实现。
+ 通过组合而非继承的方式，拓展ContextImpl的功能，在运行时选择不同的装饰类，实现不同的功能。

可以看到Activity、Service和Application都间接的继承自Context，因此我们可以通过简单的计算得出一个程序进程中有多少个Context：Context数 = Activity数 + Service数 + Application数 = Activity数 + Service数 + 1

#### Activity Context的创建过程



#### 引用

+ http://gityuan.com/2017/04/09/android_context/
+ 《Android进阶解密》



