---
layout: post
keywords: Start
description: Android中Activity的启动方式
title: Android中Activity的启动方式
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 引言

Activity的启动方式默认是standard，那我们用默认方式就好了，但是为什么Android提供了四种启动方式，基于“存在即合理的”的前提。谷歌的工程师也不是傻啊！肯定是满足了各种各样的需要。

#### 标准模式(standard)

不用任何设置你已经完成了，OK了。没什么好讲的了？如果一个activity的模式是标准，通过startActivity()方式启动的activity都会放在当前activity的上面。那什么是上面呢？

这里面就有一个task的概念。一组的activity(可能不是来自同一个应用)组成的的集合就算是一个task，好吧，这样讲还是比较南理解，我们假设分享一个东西到微信，返回回到原来的应用，那么微信的某一个页面和原来的应用可能是同一个task。

我们可以通过这种方式查看activity的task 情况：
**adb shell dumpsys activity activities **

如果理解了task的概念那么

#### singleTop

就是，在同一个task中如果在栈顶的activity与要出现的activity是同一个activity，那么就不会oncreate一个新的，还是用原来的，通过回调onNewIntent()，来获得新的Intent，需要setIntent来更新，显示新的数据。这样的好处就是，避免了Activity创建的资源损耗。但是我们通过google搜索发现，很很多人在讨论一个问题，为什么有时候不会回调到onNewIntent，这个方法，导致页面无法刷新。如果你也有同样的问题，看看manifest中是否设置，然后早startActivity前setFlag(Intent.FLAG_ACTIVITY_SINGLE_TOP)。

#### singleInstance

一个task有且自由一个该Activity。

#### singleTask

如果这个task已经拥有了这个activity，那么会把这个task以上的activtiy出栈，让该activity在栈顶，同时也会走onNewIntent。

#### 一些要点

1.另外在StartActivity的时候，还可以在Intent中指定Flag, 这些Flag有：

* FLAG_ACTIVITY_NEW_TASK, 同singleTask.
* FLAG_ACTIVITY_SINGLE_TOP, 同singleTop.
* FLAG_ACTIVITY_CLEAR_TOP, 如果Activity已经启动并且在任务栈中了，就会将其上面的所有Activity销毁，将这个Activity置于最上面，也会调用onNewInent方法.

2.Flag比xml配置有更高的优先级

3.在实际使用中，由于可能有很多场景，下面是一些可能的建议
如果是对外开放的activity可以使用singleInstance
如果是页面查看，避免重复oncreate可以使用singleTop，或者singleTask


### 参考

[Android四种Activity的加载模式 ](http://blog.csdn.net/ghj1976/article/details/6371356)
[基础总结篇之二：Activity的四种launchMode ](http://blog.csdn.net/liuhe688/article/details/6754323/)
