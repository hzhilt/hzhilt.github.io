---
layout: post
keywords: Start
description: 彻底理解Android中的尺寸（sp，dp，px，ppi，dpi）
title: 彻底理解Android中的尺寸（sp，dp，px，ppi，dpi）
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

## Android开发中，我们是通过xml实现页面的布局文件的，由于Android开源的特性，国内外无数的手机厂商生产了无数的手机，各种形状、大小各异，那么我们有必要去了解Android中的尺寸，这样我们就可以更好的适配各种奇葩的机型。

### dp（dip），在Android开发中用的最多的尺寸单位，与屏幕密度相关。

### dpi 屏幕密度：表示每英寸有多少个显示点，(x^2 + y^2)开方/寸

屏幕密度：表示每英寸有多少个显示点，与分辨率是两个不同的概念。单位是dpi（dot per inch）。
dpi的计算：dpi=Diagonal pixel/ Screen size
Diagonal pixel表示对角线的像素值

比如：计算WVGA（800*480）分辨率，3.7英寸
DPI=sqrt{800^{2} +480^{2}}/3.7
=933/3.7=252

### px pixels(像素)。 不同设备显示效果相同，一般我们HVGA代表320x480像素，这个用的比较多

据px = dp * density / 160，则当屏幕密度为160时，px = dp
根据 google 的建议，TextView 的字号最好使用 sp 做单位，而且查看TextView的源码可知Android默认使用sp作为字号单位。

### ppi(pixels per inch)是图像分辨率的单位，图像ppi值越高，画面的细节就越丰富，因为单位面积的像素数量更多、

### sp 用于文字大小

## 现在我们讲讲适配

或许你会想，我一个程序跑在市面上所有机子都能完美适配，那只是一个美好的愿望。
首先我们要准备好一些基本的工具，可以使用Android Studio自带的，也可以使用hierarchyviewer来测量。
假设现在我们要从xxhdpi（1080）的机型移植到xhdpi（720）的机型，那么原来一行显示的内容可能要一行多才能显示的完。
原来Button距离左边是20dp 即60px，现在同样20dp，现在只用20px，会在显示效果上有些差异。因此需要交互重新调整交互图。实现适配。

参考

[Supporting Multiple Screens](http://developer.android.com/guide/practices/screens_support.html)
[扒一扒那些px、pt、ppi、dpi、dp、sp之间的关系](http://blog.jobbole.com/92179/)