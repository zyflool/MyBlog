---
title: Android与H5交互
date: 2020-04-18 16:59:51
tags: 
	- Android
	- webview
categories: 学习
---

### 引入

微信，QQ空间等大量软件都内嵌了H5，不得不说是一种趋势。Android与H5互调可以让我们的实现混合开发，至于混合开发就是在一个App中内嵌一个轻量级的浏览器，一部分原生的功能改为Html5来开发。 
使用H5实现的功能能够在不升级App的情况下动态更新，而且可以在Android或iOS的App上同时运行，节约了成本，提高了开发效率。 
原理其实就是Java代码和JavaScript之间的调用。

<!--more-->

#### 一点点的网页编写知识

+ HTML 定义网页的内容
+ CSS 规定网页的布局
+ JavaScript 对网页行为进行编程

h5文档的基本结构（h5是最新的html标准，并不是一种新的语言）：

```html
<!DOCTYPE html>
<html>
  <head>
    .....
  </head>
  
  <body>
    .....
  </body>
</html>
```
一个关于JavaScript和css使用的例子：
```html
<!DOCTYPE html>
<html>
  <head>
    ......
    <script>
    	function myFunction() {
      	document.getElementById("demo").innerHTML = "段落被更改。";
    	}
  	</script>
  	<meta charset="UTF-8">
  	<link rel="stylesheet" type="text/css" href="mystyle.css">
  	<title>Title of the document</title>
	</head>

	<body>
  	<p class="content">
    	......
  	</p>
  	<button type="button" onclick="myFunction()">试一试</button>
</body>
</html>
```

### 关于Webview

#### 加载方法：

+ loadData
+ loadDataWithBaseURL
+ loadUrl

#### websettings

管理WebView的设置状态的类。

+ 获取websetting：`WebSettings websettings = mWebview.getSettings(); ` 
+ 设置支持Js：`websettings.setJavaScriptEnabled(true);`
+ 设置DOM storage缓存：`websettings.setDomStorageEnabled(true);`（有时候网页需要自己保存一些关键数据，这个时候就需要用到LocalStorage这样的东西了，WebView默认是无法使用的，所以使用的时候一定要设置这个属性）

#### WebViewClient 

WebViewClient主要用来辅助WebView处理各种通知、请求等。

```java
mWebview.setWebViewClient(new WebViewClient() {
  @Override
  public void onPageStarted(WebView view, String url, Bitmap favicon) {
    //开始载入页面调用，通常我们可以在这设定一个loading的页面，告诉用户程序在等待网络响应。
    super.onPageStarted(view, url, favicon);
  }
  
  @Override
  public void onPageFinished(WebView view, String url) {
    //在页面加载结束时调用。同样道理，我们知道一个页面载入完成，于是我们可以关闭loading条，切换程序动作。
    super.onPageFinished(view, url);
    view.loadUrl("javascript:loadContent('" + mProgress.getContent() + " ');");
  }
  
  @Override
  public boolean shouldOverrideUrlLoading(WebView view, String url) {
    //在网页跳转时调用，比如我们读取到某些特殊的URL，就可以不打开地址，取消这个操作，进行预先定义的其他操作。
    super.shouldOverrideUrlLoading(view, url);
});
```

#### WebChromeClient

WebChromeClient主要用来辅助WebView处理Javascript的对话框、网站图标、网站标题以及网页加载进度等，里面包含了很多的方法来进行操作，不同的功能使用不同的方法，没有基本的设置。

设置方法：`mWebView.setChromeClient();`

+ 当网页要启动文件选择器从 Android 设备中选择图片和文件

  需要重写onShowFileChooser方法，配合onActivityResult来获得文件（显而易见，当我们离开当前的网页去文件选择器时，activty会暂停，这个地方可能会涉及到一些生命周期的问题需要考虑）
  
+ 当网页需要获取定位的时候
  
  重写WebChromeClient的`onGeolocationPermissionsHidePrompt()`、`onGeolocationPermissionsShowPrompt()`方法
  
+ 监听网页加载进度 `onProgressChanged(WebView view, int newProgress)`

+ 监听网页图标 `onReceivedIcon(WebView view, Bitmap icon)`

+ ......
### 交互

#### Java调用H5

##### 对于没有返回值的方法

比如这里有一个Javascript方法：

```javascript
function javaCallJs(arg){
  document.getElementById("content").innerHTML =("欢迎："+arg );
}
```

在Java代码中：

```java
mWebView.loadUrl("javascript:javaCallJs("+"'"+name+"'"+")");
```

##### 对于有返回值的方法

比如这里有一个Javascript方法：

```javascript
function getContent(){
  return (this.state.content)
}
```

在Java代码中：

```java
mWebView.evaluateJavascript("javascript:this.getContent()", new ValueCallback<String>() {
  @Override
  public void onReceiveValue(String value) {
    mContent = value;
  }
});
```

#### JavaScript调用Java

##### 在Java代码中的设置

1. 实现JavaScript接口类

```java
class JS {
  @JavascriptInterface//JavaScript可以调用的方法的注解，一定要加上！！！
  public String getEdits() {
    return content;
  }
}
```

2. 配置Javascript接口

```java
mWebView.addJavascriptInterface(new JS(),"android");
```

##### 在JavaScript中使用

```javascript
const init = window.android.getEdits();
```

### 调试

在h5中被Android调用的方法只有在运行的时候才能发现是否正确，但是h5中所输出的日志并不会记录到Androidstudio的日志中，所以为了在Android上调试网页，需要另外一种方法。

1. 首先，要在WebView页面打开可以debug的设置。

```java
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
  mWebVeiw.setWebContentsDebuggingEnabled(true);
}
```

2. 在同一台电脑上的Chrome打开这个网址：Chrome://inspect：

![devtools](https://i.loli.net/2020/04/18/jYsNuy3iMSeJqw2.png)

3. 开始调试，打开你需要调试的webview界面，你会看到：

![inspect](https://i.loli.net/2020/04/18/uJ58Yc6m1VzsRoA.png)

4. 点击页面下的inspect，就可以实时看到手机上WebView页面的显示状态了。

![调试界面](https://i.loli.net/2020/04/18/CtJHNqAIemlGavK.png)

### 其他问题

#### 跨域

关于什么是跨域可以看[这个博客](https://blog.csdn.net/zhaoxy_thu/article/details/17640183)的解释。

这个问题看起来很难但是解决很简单，只需要在调用loadUrl之前：

```java
try {//跨域设置
  Class<?> clazz = mWebView.getSettings().getClass();
  Method method = clazz.getMethod(
    "setAllowUniversalAccessFromFileURLs", boolean.class);//利用反射机制去修改设置对象
  if (method != null) {
    method.invoke(mWebView.getSettings(), true);//修改设置
  }
} catch (IllegalArgumentException e) {
  e.printStackTrace();
} catch (NoSuchMethodException e) {
  e.printStackTrace();
} catch (IllegalAccessException e) {
  e.printStackTrace();
} catch (InvocationTargetException e) {
  e.printStackTrace();
}
```

#### 字符数据的处理

html文档的一些字符是没有含义的，是格式需要，但是在一定的情况下（至少我遇到的是这样的情况）这些字符会自动进行转义，所以我们需要在本地再进行转换。

```java
//unicode转换
strings[1] = strings[1].replaceAll("\\\\u003C", "<");
strings[1] = strings[1].replaceAll("\\\\n", "\n");
```

#### 网页获取页面大小

在onCreate函数中，我们通常都调用setContentView来设置布局文件，此时Android系统就会读取布局文件,但是视图此时并没有加载到Window上，并且也没有进入自己的生命周期。只有等到Activity进入resume状态时，它所拥有的View才会加载到Window上，并进行测量，布局和绘制。

一般的网页都会在加载之前测量所给容器的界面大小再进行绘制，所以为了保证网页获取你的webview的大小是正确的，需要在确保**webview绘制完毕**之后，并且**不会在暂停重启之后重新加载**，再去调用loadUrl去加载网页。

但是在onResume中加载，也会出现一定的概率获取大小错误，没有办法控制这些代码执行的先后顺序。

所以我最后使用了View的post方法，把加载网页加入到UI事件队列的末尾，这样就能保证在绘制完成后进行加载网页的操作。这里涉及到了一些handler的相关内容，可以等学习了之后再去深究。

#### Java调用多个Javascript方法

当我们在Java中调用多个Javascript方法时，这些调用其实都是在子线程完成的，所以当我们需要使用这些方法的返回值的时候，就需要加一些控制来保证使用时这些值不是空的。

所以我就使用了RxJava的zip方法来合并这些方法的调用，再一起进行处理。