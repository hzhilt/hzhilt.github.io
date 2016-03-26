---
layout: post
keywords: Start
description: 从Android的Handler讲起
title: 从Android的Handler讲起
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 前言

Android实际开发中我们我们经常会遇到一些耗时的操作，比如网络下载，查询数据库等等在其他线程，load到数据后需要主线程刷新页面。我们有那些方法呢？第一种，发送一个broadcast，接受到广播刷新，相对耗资源，需要不停的监听广播；第二种eventbus，通过事件总线通知，目前是一种流行的实现方式；第三种通过Android的handler机制实现跨线程的消息通知，使用比较广泛的实现方式。

### Handler是什么鬼？

查阅Android源码我们可以看到：

>  A Handler allows you to send and process {@link Message} and Runnable objects associated with a thread's {@link MessageQueue}.  Each Handler instance is associated with a single thread and that thread's message queue.  When you create a new Handler, it is bound to the thread /message queue of the thread that is creating it -- from that point on,it will deliver messages and runnables to that message queue and execute them as they come out of the message queue.

简单解释一下就是：Handler可以处理跨线程之间的消息，每个Handler拿到该线程的消息队列，处理队列里面的消息。我们可以这样理解，handler就想一个邮局站，也发信，收到信件处理。那么问题就来了，我们还学要邮递员和信箱才可以完成消息的投递啊。
大概是这个样子，懒得画图了。
![](http://i12.tietuku.com/0b68149320aa9637.jpg)

接下来我们继续介绍Message，MessageQueue，Looper
首先我们看看Handler的常见用法：

```
private Handler mHandlerImage = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_ICON:
                    mIcons = (List<Icon>) msg.obj;
                    mListView.setAdapter(myAdapter);
                    break;
                case MESSAGE_UPDATE:
                    Image icon = (Image) msg.obj;
                    icon.imageView.setImageDrawable(icon.drawable);
                    break;
                case MESSAGE_ERROR:
                    Toast.makeText(LoadImageActivity.this, "can not connect to the server", Toast.LENGTH_LONG).show();
                    break;
                case MESSAGE_NOTIFYLIST:
                    myAdapter.notifyDataSetChanged();
                    break;
            }
        }
    };
```

如果在主线程可以直接实现Handler的handleMessage(Message msg),如果在其他线程需要Looper.prepare(),绑定线程与looper和message，hangler。

那么线程如何发送消息呢？我们可以

```
        Thread iconThread = new Thread() {
            @Override
            public void run() {
                String string = loadData.getResult();
                List<Icon> icons = new ArrayList<Icon>();
                try {
                    JsonData jsonData = new JsonData(string, "value", "data");
                    icons = jsonData.getData();
                    Log.i("here", "get json");
                } catch (JSONException e) {
                    e.printStackTrace();
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }

                Message message = new Message();
                message.what = MESSAGE_ICON;
                message.obj = icons;
                mHandlerImage.sendMessage(message);
            }
        };
```

这样简单的就可以实现了线程之间的消息通知了，是不是很简单啊？那么你肯定会问那他是如何实现的，主线程可以给线程发消息吗？，其他线程之间可以发送消息吗？Handler只能刷新页面吗？你怎么那么多问题啊？下面我们一个一个处理

### 实现原理

线程sendMessage()之后发生了什么呢？来我们看看源码：
看到handler
![](http://i5.tietuku.com/4324e224ea55635d.png)
最后都会调到

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

我们看看

```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

msg.tatget = this;//把msg的目标处理者设置为当面的handler，然后把msg加入到当前的队列里面。

我们接着找 MessageQueue下

```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

加了一个同步锁，msg标记为** FLAG_IN_USE**

把消息加入到消息队列时，分两种情况，一种当前消息队列为空时，这时候应用程序的主线程一般就是处于空闲等待状态了，这时候就要唤醒它，另一种情况是应用程序的消息队列不为空，这时候就不需要唤醒应用程序的主线程了，因为这时候它一定是在忙着处于消息队列中的消息，因此不会处于空闲等待的状态。

第二种情况相对就比较复杂一些了，前面我们说过，当往消息队列中发送消息时，是可以指定消息的处理时间的，而消息队列中的消息，就是按照这个时间从小到大来排序的，因此，当把新的消息加入到消息队列时，就要根据它的处理时间来找到合适的位置，然后再放进消息队列中去

现在，队列里面已经有了Message了，那么如何把消息给到Handler处理呢？

我们看看Looper的代码

```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```

重点看msg.target.dispatchMessage(msg);,还记得之前我们上面说过的msg.tatget = this;了吗？，其实这个时候就是调到handler的dispatchMessage(msg)

Handler.java

```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

如果callback != null 就 handleCallback(msg);else
就是我们自己实现的 handleMessage(msg);

ok了。其实里面就是message在流动，最后被handler处理了。

### 回答上面的一些问题，

* 主线程发送消息给线程
其实就是线程需要拿到主线程的looper中的message，以下是个例子

```
private Handler handler;
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        btnTest = (Button)this.findViewById(R.id.btn_01);
        textView = (TextView)this.findViewById(R.id.view_01);
        //启动线程
        new MyThread().start();
        btnTest.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View arg0) {
                //这里handler的实例化在线程中
                //线程启动的时候就已经实例化了
                Message msg = handler.obtainMessage(1,1,1,"主线程发送的消息");
                handler.sendMessage(msg);
            }
        });
    }
    class MyHandler extends Handler{
        public MyHandler(Looper looper){
            super(looper);
        }
        public void handleMessage(Message msg){
            super.handleMessage(msg);
            textView.setText("我是主线程的Handler，收到了消息："+(String)msg.obj);
        }
    }
    class MyThread extends Thread{
        public void run(){
            Looper.prepare(); //创建该线程的Looper对象，用于接收消息
            //注意了：这里的handler是定义在主线程中的哦，呵呵，
            //前面看到直接使用了handler对象，是不是在找，在什么地方实例化的呢？
            //现在看到了吧？？？呵呵，开始的时候实例化不了，因为该线程的Looper对象
            //还不存在呢。现在可以实例化了
            //这里Looper.myLooper()获得的就是该线程的Looper对象了
            handler = new ThreadHandler(Looper.myLooper());
            //这个方法，有疑惑吗？
            //其实就是一个循环，循环从MessageQueue中取消息。
            //不经常去看看，你怎么知道你有新消息呢？？？
            Looper.loop();
        }
        //定义线程类中的消息处理类
        class ThreadHandler extends Handler{
            public ThreadHandler(Looper looper){
                super(looper);
            }
            public void handleMessage(Message msg){
                //这里对该线程中的MessageQueue中的Message进行处理
                //这里我们再返回给主线程一个消息
                handler = new MyHandler(Looper.getMainLooper());
                Message msg2 = handler.obtainMessage(1,1,1,"子线程收到:"+(String)msg.obj);
                handler.sendMessage(msg2);
            }
        }
    }
```

结果是：我是主线程的Handler，收到了消息：子线程收到:主线程发送的消息。

* Handler在开发中ide会报这样的一个提示

```
public class SampleActivity extends Activity {
 
  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }
}
```

This Handler class should be static or leaks might occur (null) less。
Since this Handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler; Make all references to members of the outer class using the WeakReference object.

可能会导致内存泄漏的意思，具体我们看看原因：
1.当一个Android应用启动的时候，会自动创建一个供应用主线程使用的Looper实例。Looper的主要工作就是一个一个处理消息队列中的消息对象。在Android中，所有Android框架的事件（比如Activity的生命周期方法调用和按钮点击等）都是放入到消息中，然后加入到Looper要处理的消息队列中，由Looper负责一条一条地进行处理。主线程中的Looper生命周期和当前应用一样长。

2.当一个Handler在主线程进行了初始化之后，我们发送一个target为这个Handler的消息到Looper处理的消息队列时，实际上已经发送的消息已经包含了一个Handler实例的引用，只有这样Looper在处理到这条消息时才可以调用Handler#handleMessage(Message)完成消息的正确处理。

3.在Java中，非静态的内部类和匿名内部类都会隐式地持有其外部类的引用。静态的内部类不会持有外部类的引用。

要解决这种问题，思路就是不适用非静态内部类，继承Handler时，要么是放在单独的类文件中，要么就是使用静态内部类。因为静态的内部类不会持有外部类的引用，所以不会导致外部类实例的内存泄露。当你需要在静态内部类中调用外部的Activity时，我们可以使用弱引用来处理。另外关于同样也需要将Runnable设置为静态的成员属性。注意：一个静态的匿名内部类实例不会持有外部类的引用。

理想的做法是

```
public class SampleActivity extends Activity {
 
  /**
   * Instances of static inner classes do not hold an implicit
   * reference to their outer class.
   */
  private static class MyHandler extends Handler {
    private final WeakReference<Sampleactivity> mActivity;
 
    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<Sampleactivity>(activity);
    }
 
    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }
 
  private final MyHandler mHandler = new MyHandler(this);
 
  /**
   * Instances of anonymous classes do not hold an implicit
   * reference to their outer class when they are static.
   */
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() { /* ... */ }
  };
 
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
 
    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 1000 * 60 * 10);
     
    // Go back to the previous Activity.
    finish();
  }
}
```

### 参考
[Android Handler机制](http://blog.csdn.net/stonecao/article/details/6417364)
[Android 异步消息处理机制 让你深入理解 Looper、Handler、Message三者关系 ](http://blog.csdn.net/lmj623565791/article/details/38377229)
[Android应用程序消息处理机制（Looper、Handler）分析 ](http://blog.csdn.net/luoshengyang/article/details/6817933)
[Android的消息循环机制 Looper Handler类分析](http://www.cnblogs.com/mengdd/p/3601294.html)
[Android开发艺术探索-Android的消息机制](http://blog.csdn.net/singwhatiwanna/article/list/2)
[Android 中Message,MessageQueue,Looper,Handler详解+实例 ](http://blog.csdn.net/yhb5566/article/details/7342786)