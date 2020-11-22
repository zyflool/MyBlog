---
title: 反编译apk文件
date: 2020-07-24 11:24:35
tags: 
	- Android
	- apk
categories: 学习
---

### APK的内部结构

APK是AndroidPackage的缩写，即Android安装包(apk)。apk安装包实际上是一个zip压缩包。我们用解压软件可以将它解压，解压后可以看到如下图的文件结构和目录结构。

![](https://s1.ax1x.com/2020/07/25/UzKDPK.png)

<!--more-->

#### classes.dex

classes.dex就是程序中java文件被编译后生成的字节码。因为Dalvik虚拟机是在Java虚拟机进行了优化，执行的是Dalvik字节码，这些Dalvik字节码是由Java字节码转换而来，一般情况下，Android应用在打包时通过AndroidSDK中的dx工具将Java字节码转换为Dalvik字节码。

#### res/
res中存放的的这个应用会使用到的资源文件，例如字符串、图片、界面布局等等。它的目录结构与Android Studio中我们看到的项目工程的res目录一样。这里面的xml文件内容都是被编译器编译过的，实际它们上已经变成了二进制文件，不再是文本文件了。

#### AndroidManifest.xml
AndroidManifest.xml也是被编译器处理过的文件，也是二进制文件，不再是文本文件。

#### resources.arsc
用来记录资源文件和资源ID之间的映射关系，用来根据资源ID寻找资源。Android的开发是分模块的，res目录专门用来存放资源文件，当在代码中需要调用资源文件时，只需要调用findviewbyId()就可以得到资源文件，每当在res文件夹下放一个文件，aapt就会自动生成对应的ID保存在.R文件，我们调用这个ID就可以，但是只有这个ID还不够，.R文件只是保证编译程序不报错，实际上在程序运行时，系统要根据ID去寻找对应的资源路径，而resources.arsc文件就是用来记录这些ID和资源文件位置对应关系的文件。

#### META-INF/

也就是一个 manifest ，从 java jar 文件引入的描述包信息的目录

#### kotlin/

这些文件包含用于标准（“内置”）Kotlin类声明的数据，这些声明未编译为`.class`文件，而是映射到平台上的现有类型。例如，`kotlin/kotlin.kotlin_builtins`包含用于在包非物理类的配置信息`kotlin`：`Int`，`String`，`Enum`，`Annotation`，`Collection`，等。

#### assets/

用于存放需要打包到APK中的静态文件。assets目录支持任意深度的子目录，用户可以根据自己的需求任意部署文件夹架构。而且res目录下的文件会在.R文件中生成对应的资源ID，assets不会自动生成对应的ID，访问的时候需要AssetManager类。

#### libs/

一般存放应用程序依赖的native库文件，一般是用C/C++编写。

### apk文件的反编译

#### 工具软件



apktool工具：获取需要反编译APK文件资源文件（图片和布局文件）。

dex2jar工具：将APK反编译成源代码 。

jd-gui工具：查看APK中源代码文件 。

#### 具体步骤

1. 使用命令行对apk文件进行资源文件的获取：

   `apktool d xxx.apk`

   反编译完成后，在apktool中会新出现一个新的文件夹，名字跟apk的名字一样的文件夹，里面就有我们需要的布局文件和图片资源文件。文件夹如下图所示：

   ![](https://s1.ax1x.com/2020/07/25/Uz35jO.png)

   2. 反编译源代码：首先将要反编译的APK后缀名改为 .zip，然后解压成文件夹，将文件夹中的classes.dex文件，然后把这个文件放在dex2jar文件夹的目录下，跟dex2jar.bat同一级目录下。然后在这个文件夹中使用命令行进行反编译：

      ` sh d2j-dex2jar.sh classes.dex`

      执行完成后在目录中会生成一个classes_dex2jar.jar的文件，代表反编译成功。

      

   3. 最后使用jd-gui工具，把上一步生成的classes_dex2jar.jar文件拖进来就能够看到源代码了

![](https://s1.ax1x.com/2020/07/25/UzGpz6.png)

**引用**

https://blog.csdn.net/linghu_java/java/article/details/7650802

https://blog.csdn.net/anddlecn/java/article/details/50972518

https://blog.csdn.net/zengrun1992/java/article/details/40076767

https://www.cnblogs.com/zhaijiahui/p/6916556.html