---
layout: post
keywords: Start
description: 从微信红包聊起
title: 从微信红包聊起
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 导言

这两年微信支付通过微信红包一炮而红，席卷大江南北，一度有超越支付宝的势头。
我们过年的时候疲于各个微信群中抢红包，很多时候，打开群，红包早就被抢夺一空，如果能过自动抢红包就好了。

### 抢红包的逻辑

1.有人在群里面发送红包
2.点击红包区域
2.1如果没有抢完，领取红包，关闭，回到群中
2.2如果抢完了，就没有了，关闭，回到群中
3.等待下次红包出现

### 语言转换

如何把界面逻辑转换成语言逻辑呢？
1.有人发红包 = 页面发生变化，出现“领取红包”字样
2.点击                = 获取焦点，自动实现click事件
2.1 还有红包  = 再次获取焦点，自动实现click事件，back事件
2.2 没有红包  = back事件

### 我们来看看如何实现

#### 首先看看 Accessibility 和 service

可以参考Android官网[Accessibility](http://developer.android.com/guide/topics/ui/accessibility/index.html)


 > An accessibility service runs in the background and receives callbacks by the system when {@link AccessibilityEvent}s are fired. Such events denote some state transition in the user interface, for example, the focus has changed, a button has been clicked,etc. Such a service can optionally request the capability for querying the content of the active window. Development of an accessibility service requires extending this class and implementing its abstract methods.

AccessibilityService 服务在后台运行，等待系统在发生 AccessibilityEvent 事件时回调。这些事件指的是用户界面上发生的状态变化， 比如焦点变更、按钮按下等等。服务可以请求“查询当前窗口中内容”的能力。 开发辅助服务需要继承该类并实现其抽象方法。
简单解释一下，当屏幕出现变化的时候，该服务可以捕捉到这些信息，可以用于各种操作，例如盲人的语音播报，辅助阅读等。
有了这个东西我们就可以很好的捕捉到各种信息。
这个时候小米开源了他的抢红包软件[小米红包](https://github.com/XiaoMi/LuckyMoneyTool)

#### 程序实现

后先配置AccessibilityService
如下：

```
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/app_name"
    android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
    android:accessibilityFeedbackType="feedbackAllMask"
    android:packageNames="com.tencent.mm"
    android:notificationTimeout="0"
    android:accessibilityFlags=""
    android:canRetrieveWindowContent="true"/>
```

然后继承Service

重写
```
public void onAccessibilityEvent(AccessibilityEvent event){
	for{
	监听窗口变化
	判断是否为红包
	红包去重
	点击红包	
	}
}
```

参考
[微信抢红包插件](https://github.com/geeeeeeeeek/WeChatLuckyMoney/tree/stable)