---
layout: post
keywords: Start
description: Android中CoordinatorLayout的用法
title: Android中CoordinatorLayout的用法
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

## Android中CoordinatorLayout的用法

我们可以看到一些很常用的Android App 已经使用上了Android5.0的特性。
个人比较喜欢的应用中，印象笔记的设置界面，以及豌豆夹的首页都使用了该特性。这么炫酷的界面一定很难吧？

很幸运，Goolge帮我做了这个事情，我们只要专注于业务逻辑就好了。来，我们看看如何实现。

首先建立一个project（本文基于Android studio ，用Eclipse的同学还是赶快来到AS的大家庭吧。），导入三个库

```
compile 'com.android.support:appcompat-v7:22.1.1'
compile 'com.android.support:design:22.2.0'
compile 'com.android.support:cardview-v7:22.2.0'
```

接下来我们看看我们的布局情况。

```
<?xml version="1.0" encoding="utf-8"?>
<!--看这里，平常我们使用LinearLayout，改为CoordinatorLayout，而且其子布局一定是AppBarLayout-->
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <!--第一部分：伸缩工具栏-->
    <android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:fitsSystemWindows="true"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <!--折叠的Toolbar布局，scrollflags scroll(可以滑动) exitUntilCollapsed(向上滚动时收缩View，但可以固定Toolbar一直在上面),里面包含了一个Image 和toolbar。contentScrim - 设置当完全CollapsingToolbarLayout折叠(收缩)后的背景颜色-->
        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/collapsing_toolbar"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:expandedTitleMarginEnd="64dp"
            app:expandedTitleMarginStart="48dp"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">

            <!--layout_collapseMode (折叠模式) :parallax - 设置为这个模式时，在内容滚动时，CollapsingToolbarLayout中的View（比如ImageView)也可以同时滚动，实现视差滚动效果，通常和layout_collapseParallaxMultiplier(设置视差因子)搭配使用。-->
            <ImageView
                android:id="@+id/backdrop"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:fitsSystemWindows="true"
                android:scaleType="centerCrop"
                android:src="@drawable/blue"
                app:layout_collapseMode="parallax" />

            <!--pin -  设置为这个模式时，当CollapsingToolbarLayout完全收缩后，Toolbar还可以保留在屏幕上。-->
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_collapseMode="pin"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />

        </android.support.design.widget.CollapsingToolbarLayout>

    </android.support.design.widget.AppBarLayout>

    <!--第二部分：主要内容，NestedScrollView和ScrollView基本功能一致，只不过NestedScrollView可以兼容新的控件-->
    <android.support.v4.widget.NestedScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical"
            android:paddingTop="24dp">

            <!--卡片布局-->
            <android.support.v7.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_margin="@dimen/card_margin">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Info"
                        android:textAppearance="@style/TextAppearance.AppCompat.Title" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="text" />

                </LinearLayout>

            </android.support.v7.widget.CardView>

            <android.support.v7.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="@dimen/card_margin"
                android:layout_marginLeft="@dimen/card_margin"
                android:layout_marginRight="@dimen/card_margin">

                <LinearLayout

                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Friends"
                        android:textAppearance="@style/TextAppearance.AppCompat.Title" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="text" />

                </LinearLayout>

            </android.support.v7.widget.CardView>

            <android.support.v7.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="@dimen/card_margin"
                android:layout_marginLeft="@dimen/card_margin"
                android:layout_marginRight="@dimen/card_margin">

                <LinearLayout

                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Related"
                        android:textAppearance="@style/TextAppearance.AppCompat.Title" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="text" />

                </LinearLayout>

            </android.support.v7.widget.CardView>

        </LinearLayout>

    </android.support.v4.widget.NestedScrollView>

    <!--第三部分：漂浮按钮-->
    <android.support.design.widget.FloatingActionButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="20dp"
        android:clickable="true"
        android:src="@drawable/blue"
        app:layout_anchor="@id/appbar"
        app:layout_anchorGravity="bottom|right|end" />
</android.support.design.widget.CoordinatorLayout>
```

最后就是Activity了

```
        //给页面设置工具栏
        final Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        getSupportActionBar().setDisplayHomeAsUpEnabled(true);

        //设置工具栏标题
        CollapsingToolbarLayout collapsingToolbar = (CollapsingToolbarLayout) findViewById(R.id.collapsing_toolbar);
        collapsingToolbar.setTitle("cheeseName");
```

效果图

![](http://i13.tietuku.com/4849c45b00701156.gif)

需要注意的是

*  toolbar 导入的是  import android.support.v7.widget.Toolbar;
*  你的手机最好是Android5.0以上的系统。

参考
[Material Design之CollapsingToolbarLayout使用 ](http://blog.csdn.net/u010687392/article/details/46906657)
[CoordinatorLayout带图片伸缩工具栏](http://www.it165.net/pro/html/201506/43424.html)
[Android M新控件之AppBarLayout，NavigationView，CoordinatorLayout，CollapsingToolbarLayout的使用](http://blog.csdn.net/feiduclear_up/article/details/46514791)
