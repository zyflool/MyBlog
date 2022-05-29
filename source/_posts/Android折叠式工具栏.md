---
title: Android折叠式工具栏
date: 2020-05-16 16:36:17
tags: 
	- Android
	- 控件学习
categories: 学习
---

如题，实现一个折叠式的工具栏

<!--more-->

#### 样例

在Androidstudio中可以通过new一个ScollingActivity得到一个官方的折叠式工具栏，布局代码如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    tools:context=".ScrollingActivity">
    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/app_bar_height"
        android:fitsSystemWindows="true"
        android:theme="@style/AppTheme.AppBarOverlay">
        <com.google.android.material.appbar.CollapsingToolbarLayout
            android:id="@+id/toolbar_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"
            app:toolbarId="@+id/toolbar">
            <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_collapseMode="pin"
                app:popupTheme="@style/AppTheme.PopupOverlay" />
        </com.google.android.material.appbar.CollapsingToolbarLayout>
    </com.google.android.material.appbar.AppBarLayout>
  
    <include layout="@layout/content_scrolling" />
  
  	<com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="@dimen/fab_margin"
        app:layout_anchor="@id/app_bar"
        app:layout_anchorGravity="bottom|end"
        app:srcCompat="@android:drawable/ic_dialog_email" />
</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

![scrollingActivity.gif](https://i.loli.net/2020/05/16/pJK3lV1YXxraZSM.gif)

#### CoordinatorLayout

>CoordinatorLayout is a super-powered FrameLayout

主要有两个用途：

1. 用作应用的顶层布局管理器，也就是作为用户界面中所有 UI 控件的容器;
2. 用作相互之间具有特定交互行为的 UI 控件的容器，通过为 CoordinatorLayout的子 View 指定 Behavior， 就可以实现它们之间的交互行为。 Behavior 可以用来实现一系列的交互行为和布局变化，比如说侧滑菜单、可滑动删除的 UI 元素，以及跟随着其他 UI 控件移动的按钮等。

其实总结出来就是 CoordinatorLayout 是一个布局管理器，相当于一个增强版的 FrameLayout，但是它神奇在于可以实现它的子 View 之间的交互行为。



![CoordinatorLayout.gif](https://i.loli.net/2020/05/16/yV6LuNoaUdeBIOQ.gif)

#### AppBarLayout

>AppBarLayout是垂直的LinearLayout，可实现matieral design app bar概念的许多功能，即滚动手势。
>
>子布局应该通过提供其所需的滚动行为setScrollFlags(int)以及相关的layout xml属性：app:layout_scrollFlags。
>
>这种布局大部分被用作CoordinatorLayout的子布局。如果在其他的viewgroup下使用AppBarLayout，它大多数功能将无法正常工作。
>
>AppBarLayout还需要单独的滚动同级，以便知道何时滚动。绑定是通过`AppBarLayout.ScrollingViewBehavior`类完成的，所以要将滚动视图的行为设置为`AppBarLayout.ScrollingViewBehavior`的实例。

##### scrollFlags属性

| Layout_scrollFlags   |                                                              |
| -------------------- | :----------------------------------------------------------- |
| Scroll               | 该视图将与滚动事件直接相关地滚动。需要将该标志设置为使其他任何标志生效。如果在此之前的任何同级视图没有此标志，则此值无效。 |
| noScroll             | 禁用在视图上滚动。该标志不应与任何其他滚动标志结合使用。     |
| enterAlways          | 进入（在屏幕上滚动）时，无论滚动视图是否也在滚动，该视图都会在任何向下滚动事件上滚动。这通常称为“快速返回”模式。 |
| enterAlwaysCollapsed | 'enterAlways'的附加标志将返回的视图修改为仅最初滚动回到其折叠高度。一旦滚动视图到达其滚动范围的末尾，该视图的其余部分将被滚动到视图中。折叠后的高度由视图的最小高度定义。 |
| exitUntilCollapsed   | 退出时（滚动到屏幕之外），视图将一直滚动，直到被“折叠”为止。折叠后的高度由视图的最小高度定义。 |
| snap                 | 滚动结束后，如果视图仅部分可见，则它将被捕捉并滚动到其最接近的边缘。例如，如果视图仅显示其底部的25％，则将其完全滚动到屏幕之外。相反，如果底部的75％可见，则它将完全滚动到视图中。 |
| snapMargins          | 与“snap”一起使用的附加标志。如果设置，则视图将被对齐到其顶部和底部边缘，而不是视图本身的边缘。 |

#### CollapsingToolbarLayout

>CollapsingToolbarLayout是Toolbar的包装，它实现了折叠的AppBar。CollapsingToolbarLayout 继承自 FrameLayout，它是用来实现 Toolbar 的折叠效果，一般它的直接子 View 是 Toolbar，当然也可以是其它类型的 View。

##### collapseMode属性

| layout_collapseMode |                                                              |
| ------------------- | ------------------------------------------------------------ |
| none                | 正常显示，不进行折叠。                                       |
| parallax            | 视图将以视差方式滚动。使用layout_collapseParallaxMultiplier属性设置视差因子 0~1之间取值，当设置了parallax时配合这个属性使用，调节自己想要的视差效果 |
| pin                 | 固定在适当位置，直到到达CollapsingToolbarLayout的底部。      |

##### contentScrim属性

当Toolbar收缩到一定程度时的所展现的主体颜色。即Toolbar的颜色。

##### title属性

当titleEnable设置为true的时候，当布局完全可见时，标题会变大，但随着布局滚动到屏幕外，标题会折叠并变小。 还可以通过collapsedTextAppearance和expandedTextAppearance属性来调整标题外观。

![CollapsingToolbarLayout.gif](https://i.loli.net/2020/05/16/pfWOInoqVM7CLkA.gif)

##### expandedTitleGravity属性

指定toolbar展开时，title所在的位置。类似的还有expandedTitleMargin、collapsedTitleGravity这些属性。

##### scrimAnimationDuration属性

toolbar收缩时，颜色变化的动画持续时间。即颜色变为contentScrim所指定的颜色进行的动画所需要的时间。

#### 引用

https://www.jianshu.com/p/640f4ef05fb2

https://www.jianshu.com/p/4a77ae4cd82f

https://juejin.im/post/5b2b00dce51d4558a75e7d63

