---
title: Java回调机制
date: 2020-12-12 21:20:40
tags:	
	- 回调
	- Java
categories: 学习
---

### 什么是回调

> 在计算机程序设计中，回调函数，或简称回调（Callback 即call then back 被主函数调用运算后会返回主函数），是指通过参数将函数传递到其它代码的，某一块可执行代码的引用。这一设计允许了底层代码调用在高层定义的子程序。

<!--more-->

往往我们通过returen值了解调用函数或者方法之后的结果，但是有时我们需要在函数执行到某一状态时根据不同的状态进行不同的操作。

简单来看，这种根据判断状态进行不同操作的函数很容易实现，修改被调用函数就可以了，这没什么难度。

但是如果我们调用的是类似于sdk中的底层方法，对于这种灵活性要求较高的需求就没有办法实现了，毕竟我们没有办法修改库。

所以这个时候回调就派上用场了，回调是将函数作为参数传递给被调用函数，以便于被调用函数在不同的状态下进行我们所需要的不同的操作。

<img src="https://pic2.zhimg.com/0ef3106510e2e1630eb49744362999f8_r.jpg?source=1940ef5c" alt="回调-维基百科" style="zoom:90%;" />

OK，传递行为作参数的意图明白了，那为什么回调就叫Callback呢？

因为我们所提供的行为参数是写在函数调用者里的，所以当被调用函数执行到对应状态时，它就会“返回”到函数调用者这里去执行对应的操作，这个从被调用函数回到调用者去调用的行为就被形象的称为“Callback”。

这里还是不太明白？

打个比方，有一家旅馆提供叫醒服务，但是要求旅客自己决定叫醒的方法。可以是打客房电话，也可以是派服务员去敲门，睡得死怕耽误事的，还可以要求往自己头上浇盆水。这里，“叫醒”这个行为是旅馆提供的，相当于库函数，但是叫醒的方式是由旅客决定并告诉旅馆的，也就是回调函数。而旅客告诉旅馆怎么叫醒自己的动作，也就是把回调函数传入库函数的动作，称为**登记回调函数**。

### 在Java中使用回调

由于Java中的方法不可以作为参数传递，所以在Java中我们使用接口对象传递方法。

下面是一个例子：

有一个Manager要给他管理的一个Worker分配工作，并且Manager要在Worker完成工作后进行一些操作。

<img src="https://s3.ax1x.com/2020/12/12/rZd3VS.jpg" alt="回调-例子" style="zoom:60%;" />



```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Manager manager = new Manager();
        manager.arrangeWork();
    }
}
interface Callback {
    void onWorkFinish();
}
class Manager {
    private Worker worker;
    public Manager() {
        worker = new Worker();
    }
    
    public void arrangeWork() throws InterruptedException {
        System.out.println("manager:我要分配工作");
        worker.work(new Callback() {
            @Override
            public void onWorkFinish() {
                System.out.println("manager:不错，加工资！");
            }
        });
    }
}

class Worker {
    public void work(Callback callback) throws InterruptedException {
        System.out.println("worker:我开始工作了");
        Thread.sleep(3000);
        System.out.println("worker:我完成工作了");
        callback.onWorkFinish();
    }
}
```

类似的，我们也可以自己实现一个Android中简单的Button响应点击事件的回调代码。

### 回调地狱

#### 举个🌰

有一个Manager管理多个Worker进行工作，并且每个人的工作都是有序并且不能同时进行的。

所以就可以得到下面这样的回调顺序：

<img src="https://s3.ax1x.com/2020/12/12/rZgAyQ.jpg" alt="回调地狱-例子" style="zoom:60%;" />

代码写起来就像这样：

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Manager manager = new Manager();
        manager.arrangeWork();
    }
}
interface Callback {
    void onWorkFinish() throws InterruptedException;
}
class Manager {
    private Worker1 worker1;
    private Worker2 worker2;
    public Manager() {
        worker1 = new Worker1();
        worker2 = new Worker2();
    }
    public void arrangeWork() throws InterruptedException {
        System.out.println("manager:我要分配工作");
        worker1.work(new Callback() {
            @Override
            public void onWorkFinish() throws InterruptedException {
                worker2.work(new Callback() {
                    @Override
                    public void onWorkFinish() {
                        System.out.println("manager:所有工作全部完成了");
                    }
                });
            }
        });
        //lambda表达式
        //worker1.work(() -> worker2.work(() -> System.out.println("manager:所有工作全部完成了")));
        //五个写起来就像这样
        //worker1.work(() -> worker2.work(() -> worker3.work(() -> worker4.work(() -> worker5.work(() -> System...)))));
    }
}
class Worker1 {
    public void work(Callback callback) throws InterruptedException {
        System.out.println("worker1:我开始工作了");
        Thread.sleep(2000);
        System.out.println("worker1:我完成工作了");
        callback.onWorkFinish();
    }
}
class Worker2 {
    public void work(Callback callback) throws InterruptedException {
        System.out.println("worker2:我开始工作了");
        Thread.sleep(2000);
        System.out.println("worker2:我完成工作了");
        callback.onWorkFinish();
    }
}
```

#### ”嵌套“

回调地狱实际上指的就是回调的嵌套，一旦要使用回调实现的功能逻辑较为复杂，就会让人进入如同“地狱”般的境地。

> ps: 实际上嵌套在任何编程中都要尽量减少，或者避免，因为无论是编写还是查错都是非常复杂的。

那有什么办法可以避免这个问题呢？

有，**响应式编程**可以解决回调地狱，我们后面要学习使用的RxJava就是使用了这种编程思想。

当然，这些都是后话了，在学习RxJava之前你还需要学习什么同步、异步等等。

