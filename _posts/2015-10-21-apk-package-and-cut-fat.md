---
layout: post
keywords: Start
description: Android apk打包过程与apk的瘦身方法
title: Android apk打包过程与apk的瘦身方法
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 引言

做Android开发的都知道使用Ant 和gradle这样强大的构建程序，只要轻轻一点IDE上的小绿点，一个apk就直接生成了，好不爽快啊！

就是就这样，我们如果不去了解apk的编译打包过程很容易忽略了应用优化的可能，现在我们就了解一下apk的诞生过程吧。

由于网上已经有很多优秀的资料了，我们应该吸取精华，进行快速的学习。现在我们就简要的描述一下吧。

### android 的包结构

首先我们解压一下apk，得到类似的包结构（不同的应用会有些差异）

![](http://i13.tietuku.com/3a0ab88ab0f87d7f.png)

* classes.dex:我们需要知道class.dex是java编译生成的java字节码，只能在android平台上的dalvik虚拟机运行。

* resources.arsc:编译生成的二进制资源文（比如values文件，XML drawables 文件等）文章[Android应用程序资源的编译和打包过程分析 ](http://blog.csdn.net/luoshengyang/article/details/8744683) 详细的分析了其生成过程，十分值得学习。微信团队也做了一个工具，可以通过混淆资源文件ID得到apk容量的减少.[微信Android资源混淆打包工具](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)

* AndroidManifest.xml：基本都知道吧，在apk中的AndroidManifest.xml是经过压缩的，可以通过AXMLPrinter2工具 解开，具体命令为：**java -jar AXMLPrinter2.jar AndroidManifest.xml**

* net ：用到的一些库

* res：res目录存放资源文件。包括图片、字符串、raw文件夹下面的音频文件、各种xml文件等等。

常见的还有以下一些

* proguard.cfg：代码混淆配置文件；

* project.properties：标示APK的target sdk和依赖关系，这里的依赖关系指示的是该APK依赖到了哪些工程；

* assets：assets目录可以存放一些配置文件（比如webview本地资源、图片资源等等），这些文件的内容在程序运行过程中可以通过相关的 API获得。

* lib：lib目录下的子目录armeabi存放的是一些so文件。这个地方多讲几句，都是在开发过程中摸索出来的。eclipse在打包的时候会根据文件名的命名规则（lib ** .so）去打包so文件，开头和结尾必须分别为“lib”和“.so”，否则是不会打包到apk文件中的。

* META-INF：META-INF目录下存放的是签名信息，用来保证apk包的完整性和系统的安全。在eclipse编译生成一个apk包时，会对所有 要打包的文件做一个校验计算，并把计算结果放在META-INF目录下。这就保证了apk包里的文件不能被随意替换。比如拿到一个apk包后，如果想要替 换里面的一幅图片，一段代码， 或一段版权信息，想直接解压缩、替换再重新打包，基本是不可能的。如此一来就给病毒感染和恶意修改增加了难度，有助于保护系统的安全。

可以看到解压后的内容与开发的时候有很大的区别，目前来说有一些反编译工具可以获取到一些应用程序的信息，这个过程也是开发者与开发者之间的博弈过程，当前我们虽然鼓励开源，自由软件，过度的破译软件也会带来灾难，以后可以写一篇[软件的加固与反编译](...)

### apk的打包过程

很明显，我们应该现有打包过程再有apk资源分析，但是了解了资源后可以更快的了解其打包过程，首先我们来看一下打包过程的图解，相信这个过程也是非常容易理解的
![](http://pic002.cnblogs.com/images/2011/267603/2011122014434195.jpg)

& like this
![](http://pic002.cnblogs.com/images/2011/267603/2011122014463111.gif)
![](http://pic002.cnblogs.com/images/2011/267603/2011122014464569.gif)

其中使用到的工具

| 名称 | 功能介绍 | 在操作系统中的路径 |
| : ---- : | : ------------- : | : ------------- : |
|aapt 	|Android资源打包工具 	|${ANDROID_SDK_HOME}/platform-tools/appt
|aidl 	|Android接口描述语言转化为.java文件的工具 |	${ANDROID_SDK_HOME}/platform-tools/aidl
|javac 	|Java Compiler 	|${JDK_HOME}/javac或/usr/bin/javac
|dex 	|转化.class文件为Davik VM能识别的.dex文件 	|${ANDROID_SDK_HOME}/platform-tools/dx
|apkbuilder 	|生成apk包 	|${ANDROID_SDK_HOME}/tools/opkbuilder
|jarsigner 	|.jar文件的签名工具 	|${JDK_HOME}/jarsigner或/usr/bin/jarsigner
|zipalign 	|字节码对齐工具 	|${ANDROID_SDK_HOME}/tools/zipalign

接下来我们要分析一下里面重要的步骤

 * 第一步：打包资源文件，生成R.java文件
【输入】Resource文件（就是工程中res中的文件）、Assets文件（相当于另外一种资源，这种资源Android系统并不像对res中的文件那样优化它）、AndroidManifest.xml文件（包名就是从这里读取的，因为生成R.java文件需要包名）、Android基础类库（Android.jar文件）
【输出】打包好的资源（一般在Android工程的bin目录可以看到一个叫resources.ap_的文件就是它了）、R.java文件（在gen目录中，大家应该很熟悉了）
【工具】aapt工具

 * 第二步：处理AIDL文件，生成对应的.java文件（当然，有很多工程没有用到AIDL，那这个过程就可以省了）
【输入】源码文件、aidl文件、framework.aidl文件
【输出】对应的.java文件
【工具】aidl工具

 * 第三步：编译Java文件，生成对应的.class文件
【输入】源码文件（包括R.java和AIDL生成的.java文件）、库文件（.jar文件）
【输出】.class文件
【工具】javac工具

 * 第四步：把.class文件转化成Davik VM支持的.dex文件
【输入】 .class文件（包括Aidl生成.class文件，R生成的.class文件，源文件生成的.class文件），库文件（.jar文件）
【输出】.dex文件
【工具】javac工具

 * 第五步：打包生成未签名的.apk文件
【输入】打包后的资源文件、打包后类文件（.dex文件）、libs文件（包括.so文件，当然很多工程都没有这样的文件，如果你不使用C/C++开发的话）
【输出】未签名的.apk文件
【工具】apkbuilder工具

 * 第六步：对未签名.apk文件进行签名
【输入】未签名的.apk文件
【输出】签名的.apk文件
【工具】jarsigner

 * 第七步：对签名后的.apk文件进行对齐处理（不进行对齐处理是不能发布到Google Market的）
【输入】签名后的.apk文件
【输出】对齐后的.apk文件
【工具】zipalign工具

### apk瘦身

目前我们编译的apk包动辄几十M，通过上面的分析，我们可以知道可能的原因有，使用可大量的第三方lib，使用过多的资源，或者说高清图片等，其实一般代码占不了太多的资源。

* 代码混淆，减少dex，也可以防止代码泄漏
* 如果使用了较多的lib，可以通过自己抽离部分代码来去掉lib包
* 资源文件：通过Lint工具扫描代码中没有使用到的静态资源。
* 图片资源的优化原则是：在不降低图片效果、保证APK显示效果的前提下缩小图片文件的大小。可以使用使用tinypng优化大部分图片资源，[tinypng](https://tinypng.com) 可以减少30-50%。
* 优化你的代码

[关于APK瘦身值得分享的一些经验](http://www.open-open.com/lib/view/open1428321523479.html)

[如何给你的Android 安装文件（APK）瘦身](http://www.2cto.com/kf/201411/353176.html)

[Android应用程序（APK）的编译打包过程 ](http://blog.csdn.net/songjinshi/article/details/9059611)

[Android应用程序资源的编译和打包过程分析 ](http://blog.csdn.net/luoshengyang/article/details/8744683)
