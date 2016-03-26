---
layout: post
keywords: Start
description: 深入Android的事件分发(Touch)
title: 深入Android的事件分发(Touch)
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 引言
在很多自定义View中，大家喜欢给她加上各种特性，左滑，右滑等等，构成十分炫酷的动画，完成了一个个不可思议的构造，但是过程是曲折的。有时候程序并不能按照既定的方式去完成，明明点击却没有响应，滑动变成了点击，等等，让你一凑莫展。这个时候我们应该好好取学习一下Android的事件分发的机制。

### 我自己走过的坑

自己曾经自定义一个layout，里面有一个imageview和一个button，onTouch时，出现Button不能拖动，原来是由于button设置了clickable为true，导致了button拦截了ontouch时间。

那么touch事件与click事件是怎么的一个回事呢？

### 事件分发前置知识

Android的事件分发，是一个比较难摸透的点，如果不去了解源码，或者自己做一下实验，很难理解其中的逻辑，下面我们就大概去捋一下这里面的逻辑。
Android本事支持很多中触摸方式，例如点击，长按，多指头触摸，滑动，快速滑动，系列的操作，那么如果我们自己去实现这里面的事件分发，我们要怎么做？

* 首先我们要给每个控件设定触摸的类型（Flag），设定每个layout的触摸类型
* 然后layout的顶层到底层一致遍历每个组件，或者反方向
* 分别判断每个view是有处在当面触摸类型的Flag，如果有就执行
* 知道手指离开屏幕

我是这么想的，google工程师看到后只会说，no bb，show me the code。

好吧，那么我只能看看google的源码了

```
    /**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```

我们看到

```
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
```

这里面讲的是如果view设置了OnTouchListener,者先执行onTouch，再执行onTouchEvent()

跳到了 onTouchEvent

我们看看这里面做了什么？

```
    public boolean onTouchEvent(MotionEvent event) {
      .................................
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```

主要分析一下，如果，设置了点击或者长按的，则进入事件分发
对点击和长按的分析，可以看到performClick();再里面。

由此我们可以得到一下结论：
1、整个View的事件转发流程是：
View.dispatchEvent->View.setOnTouchListener->View.onTouchEvent
在dispatchTouchEvent中会进行OnTouchListener的判断，如果OnTouchListener不为null且返回true，则表示事件被消费，onTouchEvent不会被执行；否则执行onTouchEvent。

2、onTouchEvent中的DOWN,MOVE,UP
DOWN时：
a、首先设置标志为PREPRESSED，设置mHasPerformedLongPress=false ;然后发出一个115ms后的mPendingCheckForTap；
b、如果115ms内没有触发UP，则将标志置为PRESSED，清除PREPRESSED标志，同时发出一个延时为500-115ms的，检测长按任务消息；
c、如果500ms内（从DOWN触发开始算），则会触发LongClickListener:
此时如果LongClickListener不为null，则会执行回调，同时如果LongClickListener.onClick返回true，才把mHasPerformedLongPress设置为true;否则mHasPerformedLongPress依然为false;

MOVE时：
主要就是检测用户是否划出控件，如果划出了：
115ms内，直接移除mPendingCheckForTap；
115ms后，则将标志中的PRESSED去除，同时移除长按的检查：removeLongPressCallback();

UP时：
a、如果115ms内，触发UP，此时标志为PREPRESSED，则执行UnsetPressedState，setPressed(false);会把setPress转发下去，可以在View中复写dispatchSetPressed方法接收；
b、如果是115ms-500ms间，即长按还未发生，则首先移除长按检测，执行onClick回调；
c、如果是500ms以后，那么有两种情况：
i.设置了onLongClickListener，且onLongClickListener.onClick返回true，则点击事件OnClick事件无法触发；
ii.没有设置onLongClickListener或者onLongClickListener.onClick返回false，则点击事件OnClick事件依然可以触发；
d、最后执行mUnsetPressedState.run()，将setPressed传递下去，然后将PRESSED标识去除；

就是这样：
![](http://i4.tietuku.com/cc4506f73d29d8f5.png)

接着你就可以自己分析一下ViewGroup了。


###  Log信息
然后，我做了一下这个实验
在一个layout中只放了一个button，观察view之间的分发流程
当button onTouch return false时

>I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ button down
I/Touch﹕ Button layout onTouchEvent
I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ Button layout onTouchEvent
I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ Button layout onTouchEvent
I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ button up
I/Touch﹕ Button layout onTouchEvent
I/Touch﹕ button click

分析：父布局先分发，没有拦截，button分发，ontouch优先级比OntouchEvent高，先down，再event
接着事件分发，up ---Event。最后如果ontouch     return false，事件继续分发。响应点击事件 click。

当 onTouch return true

>I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ button down
I/Touch﹕ parent layout dispatchTouchEvent
I/Touch﹕ parent layout onInterceptTouchEvent
I/Touch﹕ Button layout dispatchTouchEvent
I/Touch﹕ button up

分析：return true之后，button 的onTouchEvent没有被调用，而且click事件也被拦截了。

### 一个例子

下面是一个天气曲线的例子，通过手指滑动切换曲线，首先想到的通过touch事件更新view上的曲线，大概是这个样子
![](http://i4.tietuku.com/9be0a053283bebff.gif)

```
public class AnimationLineView extends View {

    private static final int SIX = 6;
    private static final int TOP = 1000;

    public AnimationLineView(Context context) {
        super(context);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX, mStartY);
    }


    public AnimationLineView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX, mStartY);
    }

    public AnimationLineView(Context context, AttributeSet attrs) {
        super(context, attrs);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX, mStartY);

    }

    private int[] mStartX = new int[7];
    private int[] mStartY = new int[7];
    private int[] mLastY = new int[7];


    private final int FIRST_X = 29;


    private int mW;
    private int mH;
    private Paint mPaint;
    private Bitmap mBitmap;
    private Canvas mCanvas;

    private int WIDTH = 2;

    private Context mContext;

    private float startX = 0;
    private float endX = 0;

    private float fraction = 0;

    private boolean isLeft = true;
    private boolean isRight = false;

    private void getScreenWH() {
        DisplayMetrics dm = new DisplayMetrics();
        ((Activity) mContext).getWindowManager().getDefaultDisplay().getMetrics(dm);
        mW = dm.widthPixels;
        mH = dm.heightPixels;
    }

    public void initPaint() {
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
        mPaint.setDither(true);
        mPaint.setColor(Color.RED);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeJoin(Paint.Join.ROUND);
        mPaint.setStrokeCap(Paint.Cap.ROUND);
        mPaint.setStrokeWidth(WIDTH);
    }

    public void initCanvas() {
        setBackgroundColor(Color.WHITE);//设置背景
        mBitmap = Bitmap.createBitmap(mW, mH, Bitmap.Config.ARGB_8888);//需要显示的画布
        mCanvas = new Canvas(mBitmap);
        mCanvas.setBitmap(mBitmap);
    }

    public void initData() {

        int delta = (mW - 2 * FIRST_X) / SIX;

        for (int i = 0; i < 7; i++) {
            mStartX[i] = FIRST_X + i * delta;
        }

        mStartY = new int[]{900, 852, 950, 888, 756, 687, 800};
        mLastY = new int[]{800, 564, 897, 765, 965, 800, 600};
    }

    private void initLayout(int[] x, int[] y) {

        int[] mX = x;
        int[] mY = y;
        MyPath path = new MyPath();
        path.moveTo(0, 1000);
        path.lineTo(mH, 1000);
        mCanvas.drawPath(path, mPaint);

        path = new MyPath();

        path.moveTo(mX[0], mY[0]);

        path.myQuadTo(mX[0], mY[0], mX[1], mY[1]);
        path.myQuadTo(mX[1], mY[1], mX[2], mY[2]);
        path.myQuadTo(mX[2], mY[2], mX[3], mY[3]);
        path.myQuadTo(mX[3], mY[3], mX[4], mY[4]);
        path.myQuadTo(mX[4], mY[4], mX[5], mY[5]);
        path.myQuadTo(mX[5], mY[5], mX[6], mY[6]);

        mCanvas.drawPath(path, mPaint);

        path.reset();

        path.myCircleLineto(mX[0], mY[0], mX[0], TOP);
        path.myCircleLineto(mX[1], mY[1], mX[1], TOP);
        path.myCircleLineto(mX[2], mY[2], mX[2], TOP);
        path.myCircleLineto(mX[3], mY[3], mX[3], TOP);
        path.myCircleLineto(mX[4], mY[4], mX[4], TOP);
        path.myCircleLineto(mX[5], mY[5], mX[5], TOP);
        path.myCircleLineto(mX[6], mY[6], mX[6], TOP);

        mCanvas.drawPath(path, mPaint);
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        canvas.drawBitmap(mBitmap, 0, 0, null);//把bitmap显示出来
    }

public boolean onTouchEvent(MotionEvent event) {

        switch (event.getAction()) {
            //down 的时候记录位置
            case MotionEvent.ACTION_DOWN:

                float x = event.getRawX();
                startX = x;
                break;

            //移动判断左右方向
            case MotionEvent.ACTION_MOVE:
                float x1 = event.getRawX();
                endX = x1;

                if (endX - startX < 0 && (isLeft == true)) {
                    break;
                }

                if (endX - startX > 0 && (isRight == true)) {
                    break;
                }
                //fraction，用于做动画的速率
                fraction = Math.abs(endX - startX) / 1000;

                if (fraction > 1) {
                    fraction = 1;
                }

                if (fraction < 0)
                    fraction = 0;


                int[] newX = new int[7];
                for (int i = 0; i < 7; i++) {
                    if (isLeft == true) {
                        newX[i] = mStartY[i] + (int) ((mLastY[i] - mStartY[i]) * fraction);
                    } else {
                        newX[i] = mLastY[i] + (int) ((mStartY[i] - mLastY[i]) * fraction);
                    }
                }
                initCanvas();
                initLayout(mStartX, newX);

                break;
            //判断up的位置，继续做动画
            case MotionEvent.ACTION_UP:

                if ((endX - startX > 0) && (isRight == false)) {
                    if (fraction > 0.5) {
                        runAnimation(2000);
                        isRight = true;
                        isLeft = false;
                    }
                    if (fraction < 0.5) {
                        runAnimation(0);
                        isRight = false;
                        isLeft = true;
                    }
                }
                if ((endX - startX < 0) && (isLeft == false)) {
                    if (fraction > 0.5) {
                        backAnimation(2000);
                        isLeft = true;
                        isRight = false;
                    }
                    if (fraction < 0.5) {
                        backAnimation(0);
                        isLeft = false;
                        isRight = true;
                    }
                }
                break;
        }
        return true;
    }

    private void runAnimation(long fraction) {

        isLeft = false;
        isRight = true;

        ValueAnimator valueAnimator = new ValueAnimator();
        valueAnimator.setDuration(2000);
        valueAnimator.setObjectValues(new int[7]);
        //valueAnimator.setCurrentPlayTime(playTime);

        valueAnimator.setEvaluator(new TypeEvaluator() {
            @Override
            public Object evaluate(float fraction, Object startValue, Object endValue) {
                int[] x = new int[7];
                for (int i = 0; i < 7; i++) {
                    x[i] = mStartY[i] + (int) ((mLastY[i] - mStartY[i]) * fraction);
                }

                return x;
            }
        });
        //valueAnimator.start();

        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {

                int[] x = (int[]) animation.getAnimatedValue();
                //mStartY = x;
                initCanvas();
                initLayout(mStartX, x);
            }
        });

        valueAnimator.setCurrentPlayTime(fraction);
    }

    private void backAnimation(long fraction) {

        isLeft = true;
        isRight = false;

        ValueAnimator valueAnimator = new ValueAnimator();
        valueAnimator.setDuration(2000);
        valueAnimator.setObjectValues(new int[7]);


        valueAnimator.setEvaluator(new TypeEvaluator() {
            @Override
            public Object evaluate(float fraction, Object startValue, Object endValue) {
                int[] x = new int[7];
                for (int i = 0; i < 7; i++) {
                    x[i] = mLastY[i] + (int) ((mStartY[i] - mLastY[i]) * fraction);
                }

                return x;
            }
        });
        //valueAnimator.start();
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {

                int[] x = (int[]) animation.getAnimatedValue();
                //mStartY = x;
                initCanvas();
                initLayout(mStartX, x);
            }
        });
        valueAnimator.setCurrentPlayTime(fraction);
    }

}

```

画贝塞尔曲线

```
public class MyPath extends Path {

    public void myQuadTo(float x1,float y1,float x2, float y2){

        float x0 = (x1+x2)/2;
        float y0 = (y1+y2)/2;

        float x10 = (x0+x1)/2;
        float y10 = y1;

        quadTo(x10,y10,x0,y0);

        float x02 = (x0+x2)/2;
        float y02 = y2;

        quadTo(x02,y02,x2,y2);
    }

    public void myCircleLineto(float x1,float y1,float x2,float y2){

        addCircle(x1,y1,5f,Direction.CW);
        moveTo(x1,y1);
        lineTo(x2,y2);

    }
}

```

以上是一年多前的代码，现在回头看看，实在是渣，现在也懒得改回来，也算是自己的一次回顾吧。

通过后来的分析，其实我们可以通过viewpager轻松实现该动画

```
mViewPager.setOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

                mLineView.setCurrentPosition((long)(positionOffset*2000),position);
            }

            @Override
            public void onPageSelected(int position) {

            }

            @Override
            public void onPageScrollStateChanged(int state) {

            }
        });

```

```

public class LineView extends View {
    private static final int SIX = 6;
    private static final int TOP = 1000;

    public LineView(Context context) {
        super(context);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX,mStartY);
        initAnimation(0,1);
    }

    public LineView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX,mList.get(0));
        initAnimation(0,1);
    }

    public LineView(Context context, AttributeSet attrs) {
        super(context, attrs);
        mContext = context;
        getScreenWH();
        initPaint();
        initCanvas();
        initData();
        initLayout(mStartX,mStartY);
        initAnimation(0,1);
    }

    private int[] mStartX = new int[7];//x-axis

    private int[] mStartY = new int[7];//weatherInfo
    private int[] mLastY = new int[7];
    private int[] mNextY = new int[7];
    private int[] mEndY  = new int[7];// case null;

    private List<int[]> mList = new ArrayList<>();

    private final int FIRST_X = 29;

    private int mW ;
    private int mH ;
    private Paint mPaint;
    private Bitmap mBitmap;
    private Canvas mCanvas;

    private int WIDTH = 5;

    private Context mContext;

    private ValueAnimator mValueAnimator;

    private void getScreenWH(){
        DisplayMetrics dm = new DisplayMetrics();
        ((Activity) mContext).getWindowManager().getDefaultDisplay().getMetrics(dm);
        mW = dm.widthPixels;
        mH = dm.heightPixels;
    }

    public  void initPaint(){
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
        mPaint.setDither(true);
        mPaint.setColor(Color.GRAY);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeJoin(Paint.Join.ROUND);
        mPaint.setStrokeCap(Paint.Cap.ROUND);
        mPaint.setStrokeWidth(WIDTH);
    }

    public void initCanvas(){
        //setBackgroundColor(Color.WHITE);//设置背景
        //setAlpha(255);
        mBitmap = Bitmap.createBitmap(mW, mH, Bitmap.Config.ARGB_8888);//需要显示的画布
        mCanvas = new Canvas(mBitmap);
        mCanvas.setBitmap(mBitmap);
    }

    public void initData(){

        int delta = (mW - 2*FIRST_X)/SIX;
        for(int i=0;i<7;i++)
        {
            mStartX[i] = FIRST_X + i*delta;
        }

        mStartY = new int[]{900,852,950,888,756,687,800};
        mNextY = new int[]{850,600,900,800,860,730,650,};
        mLastY = new int[]{800,564,897,765,965,800,600};
        mEndY = new int[]{1000,1000,1000,1000,1000,1000,1000};

        mList.add(mStartY);
        mList.add(mNextY);
        mList.add(mLastY);
        mList.add(mEndY);
    }

    private void initLayout(int[] x,int[] y) {

        int[] mX = x;
        int[] mY = y;
        MyPath path = new MyPath();
        path.moveTo(0,1000);
        path.lineTo(mH,1000);
        mCanvas.drawPath(path,mPaint);

        path = new MyPath();
        path.moveTo(mX[0],mY[0]);
        // draw bezier line
        for(int i = 0;i<6;i++){
            path.myQuadTo(mX[i],mY[i],mX[i+1],mY[i+1]);
        }

        mCanvas.drawPath(path,mPaint);

        path.reset();
        // draw line between the line
        for(int j = 0;j<7;j++){
            path.myCircleLineto(mX[j],mY[j],mX[j],TOP);
        }

        mCanvas.drawPath(path,mPaint);
        invalidate();
    }


    private void initAnimation(final int start,final int last) {
        mValueAnimator = new ValueAnimator();
        mValueAnimator.setDuration(2000);
        mValueAnimator.setObjectValues(new int[7]);

        mValueAnimator.setEvaluator(new TypeEvaluator() {
            @Override
            public Object evaluate(float fraction, Object startValue, Object endValue) {
                int[] x = new int[7];
                for(int i = 0;i<7;i++)
                {
                    x[i] = mList.get(start)[i] + (int) ((mList.get(last)[i] - mList.get(start)[i]) * fraction);
                }

                return x;
            }
        });

        mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {

                int[] x = (int[])animation.getAnimatedValue();
                initCanvas();
                initLayout(mStartX,x);
            }
        });
    }

    @Override
    protected void onDraw(Canvas canvas) {
        canvas.drawBitmap(mBitmap, 0, 0, null);//把bitmap显示出来
    }



    public void setCurrentPosition(long time,int position){

        initAnimation(position,position+1);
        mValueAnimator.setCurrentPlayTime(time);


    }
}

```

viewpager本身自己处理了很多的滑动事件，我们可以利用快速开发。
Android的事件分发不可怕，只要你分清楚每个view的职能，知道他要做什么，赋予他能力，就可以做到你想做的东西。

### 参考
[Android View 事件分发机制 源码解析](http://blog.csdn.net/lmj623565791/article/details/38960443)
[Android:30分钟弄明白Touch事件分发机制](http://www.cnblogs.com/linjzong/p/4191891.html) (实际上你不可能30分钟搞定)
[《Android深入透析》之Android事件分发机制](http://www.cnblogs.com/duoduohuakai/p/3996385.html)
[
Android 编程下 Touch 事件的分发和消费机制 ](http://www.cnblogs.com/sunzn/archive/2013/05/10/3064129.html)
[Android Touch事件传递机制全面解析（从WMS到View树） ](http://blog.csdn.net/ns_code/article/details/49848801)
