---
layout: post
keywords: Start
description: 彻底理解时区（TimeZone）
title: 彻底理解时区（TimeZone）
categories: [技术]
tags: [技术]
group: archive
icon: globe
---

## 彻底理解时区（TimeZone）

说到时区，很多人都还记得中国位于东8区，然后有一个什么国际日期变更线，再有一个格林尼治什么的，基本都还给了高中老师。现在很好奇，为什么美国不是早上六点起床，九点睡觉？谁能告诉我。

说道时区这个东西，在应用程序中有着很多用处，例如你的手机切换了语言或者时区（我知道你不会去切换，你也找不到哪里切换），或者一些商务人士经常收发邮件，建立google 日历，后者翻墙连到了日本，那么电脑上的联网应用时区也可能会改变。

好吧，我知道我要处理这个问题。
首先放出一些概念

* UTC ：Coordinated Universal Time ，又称世界统一时间，世界标准时间，国际协调时间。协调世界时是以原子时秒长为基础，在时刻上尽量接近于世界时的一种时间计量系统。
* Unix时间戳（英文为Unix epoch, Unix time, POSIX time 或 Unix timestamp）是从1970年1月1日（UTC/GMT的午夜）开始所经过的秒数，不考虑闰秒。

好的，我们现在考虑一种情景，一个美国朋友邀请你一次会面，发送的时间是
：* TZID=America/Los_Angeles:20151019T063045 *，那么你在中国究竟是什么时候开会呢？这里就用到UTC了，* TZID=America/Los_Angeles:20151019T063045 *  ==  * 20151019T102045Z *  =  * TZID =Asia/Shanghai:20151019T183045 *

好的，我们现在看一下代码

```
Property property = entryComponent.getProperty(Property.TZID);
if (property != null) {
    defaultTZ = property.getValue();
}
// convert time zone
defaultTZ = ConvertWindowsTimeZone.convertTimeZone(defaultTZ);
if (TextUtils.isEmpty(defaultTZ)) {
    defaultTZ = Utils.getTimeZone(context, null);
}
```

![](http://www.timedate.cn/images/timezone.jpg)
