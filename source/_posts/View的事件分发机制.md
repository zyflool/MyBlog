---
title: View的事件分发机制
date: 2020-05-29 11:45:06
tags: 
	- Android
	- View
categories: 学习
---

### 事件——MotionEvent类

+ ACTION_DOWN 手指刚接触屏幕
+ ACTION_UP 手指从屏幕上松开
+ ACTION_MOVE 手指在屏幕上移动
+ ......

<!--more-->

一次手指触摸屏幕的行为会触发一系列的点击事件，同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以ACTION_DOWN事件开始，中间含有数量不定的ACTION_MOVE事件，最终以ACTION_UP事件结束。

### 事件分发的三个重要方法

```java
public boolean dispatchTouchEvent(MotionEvent ev) //用来分发事件

public boolean onInterceptTouchEvent(MotionEvetn event) //用来拦截事件
  
public boolean onTouchEvent(MotionEvent event) //用来处理事件
  
//三个方法的基本处理逻辑
public boolean dispatchTouchEvent(MotionEvent ev) {
  	boolean consume = false;
  	if ( onInterceptTouchEvent(ev) ) {
      	consume = onTouchEvent(ev);
    } else {
      	consume = child.dispatchTouchEvent(ev);
    }
  	return consum;
}
```

![点击事件在View中的分发.png](https://i.loli.net/2020/05/30/7ud3IoFREkDwJme.png)

### 事件分发的基本顺序

Activity-->Window-->ViewGroup/View-->...-->View

#### Activity处理点击事件

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    onUserInteraction();//空方法，用来重写以获知已进行了交互
  }
  //交给Activity所附属的Window进行事件分发
  if (getWindow().superDispatchTouchEvent(ev)) { 
    return true;
  }
  return onTouchEvent(ev); // 如果事件没有进行处理，则交给Activity的onTouchEvent
}
```

#### PhoneWindow(Window的实现类)

```java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor; //DecorView继承自FrameLayout

@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
  return mDecor.superDispatchTouchEvent(event); //将事件传递给DecorView
}
```

+ 正常情况下，一个事件序列只能被一个View拦截且消耗.
+ 某个View一旦决定拦截，后续的点击事件都不会调用此View的onInterceptTouchEvent()方法.
+ 某个View一旦不消耗ACTION_DOWN事件，整个序列就直接交给它的父元素处理.
+ 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View.
+ 当最底层的View都不消耗事件时，就一层层向外调用父元素的onTouchEvent()方法，直到到Activity层.