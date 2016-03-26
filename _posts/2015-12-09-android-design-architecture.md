---
layout: post
keywords: Start
description: 聊一聊Android的设计框架
title: 聊一聊Android的设计框架
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 前言

这个话题我可能不可以深入的进入，因为自己没有很好的把握来深入这个话题，最多算是抛砖引玉了。我们在谈论Android架构的时候更多是从一些分模块，抽象这样的方向去着手，我们的前辈就有很多好的遗产，例如MVC（Model-View-Controller）框架，这个设计框架可以直接用在Android上，也得到了广泛的应用。

但是当我们写一个Activity，一个页面的时候去实现MVC框架就会觉得很累，没必要，接下来你的代码越来越多，需求越来越多，你发现这个时候你已经无力回天了，你的Activity里面充满了各种逻辑，各种隐藏的bug，来了一个需求又在Activity中改改改，然后最难受的是你还要在里面找各种bug，你悔不当初啊！

那么有没有更好，更加轻便，适合Android的架构呢？

### MVP

说的不是NBA里面的最有价值球员，是Model,View,Presenter。MVP与MVC有着一个重大的区别：在MVP中View并不直接使用Model，它们之间的通信是通过Presenter (MVC中的Controller)来进行的，所有的交互都发生在Presenter内部，而在MVC中View会直接从Model中读取数据而不是通过 Controller。
来来来，我们看看下面的一副图比较一下

![](http://img.blog.csdn.net/20150622212916054)

这样我们就可以清晰的看到区别了，

* View 对应于Activity，负责View的绘制以及与用户交互
* Model 依然是业务逻辑和实体模型
* Presenter 负责完成View于Model间的交互

那么我们看看MVP有那些优点

* 模型与视图完全分离，我们可以修改视图而不影响模型
   Activity只做显示，我不管你下面的模型
* 可以更高效地使用模型，因为所有的交互都发生在一个地方——Presenter内部
   所有的交互逻辑都在P里面，改动不要影响别人就好
* 可测试
   可以快速的使用接口来进行测试。

### 有个例子可好

在这里直接使用鸿洋的博客案例进行分析
[浅谈 MVP in Android](http://blog.csdn.net/lmj623565791/article/details/46596109)

我叫小明写一个登录页面，他五分钟说写好了，我一看

```
public class LoginActivity extends Activity {

    private TextView mName;
    private TextView mPassword;

    private Button mLogin;
    private Button mClear;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView();

        // init
        mName = (TextView) findViewById();
        mPassword = (TextView) findViewById();
        mLogin = (Button) findViewById();

        mLogin.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String name = mName.getText().toString();
                String password = mPassword.getText().toString();

                if (TextUtils.isEmpty(name)){
                    Toast.makeText(LoginActivity.this,"name is null",Toast.LENGTH_SHORT).show();
                }else if (TextUtils.isEmpty(password)){
                    Toast.makeText(LoginActivity.this,"password is null",Toast.LENGTH_SHORT).show();
                }

                
                if (checkNameAndPassword()){
                    Toast.makeText(LoginActivity.this,"success",Toast.LENGTH_SHORT).show();
                }else {
                    Toast.makeText(LoginActivity.this,"fail",Toast.LENGTH_SHORT).show();
                }
            }
        });

        mClear.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mName.setText("");
                mPassword.setText("");
            }
        });
        
    }
    
    
    // connect the server and check 
    private boolean checkNameAndPassword(){
        return false;
    }
}
```

我擦，真的写好了，而且还真的可以耶！

那还搞什么MVC框架啊！

当然，我们前面就讲过，单一功能使用这些框架可能会得不偿失，但是我们要随着业务的增加和功能的增多，原来的框架无法扩展，只会越来越糟。

那么mvp框架应该如何搭建呢？

![](http://i4.tietuku.com/916a29c76d30f171.jpg)

具体代码查看上面的blog。

参考：

最近在逛blog的时候看到了这个
[Android项目重构之路:架构篇](http://keeganlee.me/post/android/20150605)
里面讲的也是如何搭建框架，一开始搭建框架很累，但是等到项目到了一定规模，你就开始享受框架给你带来的畅快吧，当别人还在加班解bug的时候，你已经安然入睡了。^_^