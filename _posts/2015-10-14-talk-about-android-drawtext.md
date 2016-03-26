---
layout: post
keywords: Start
description: 聊聊Android中的drawText。
title: 聊聊Android中的drawText。
categories: [Android]
tags: [Android]
group: archive
icon: globe
---

### 今天遇到一个小问题，就是在自定义view中使用canvas.drawText()函数的时候，发现文字没有水平居中，困惑。

通过查看其API如下，其中，x，y 是所画字符串的起始位置。那么现在问题就来了，其实位置究竟是左上角，右上角还是字符串中心呢？

``` 
/**
 * Draw the text, with origin at (x,y), using the specified paint. The
 * origin is interpreted based on the Align setting in the paint.
 *
 * @param text  The text to be drawn
 * @param x     The x-coordinate of the origin of the text being drawn
 * @param y     The y-coordinate of the origin of the text being drawn
 * @param paint The paint used for the text (e.g. color, size, style)
 */
 public void drawText(@NonNull String text, float x, float y, @NonNull Paint paint) {
    native_drawText(mNativeCanvasWrapper, text, 0, text.length(), x, y, paint.mBidiFlags,
                paint.mNativePaint, paint.mNativeTypeface);
 }
```

接着我们展开了实验。
首先我们设置要显示的字符串 s

```
float textWidth = mPaint.measureText(s);
mPaint.getTextBounds(s, 0, s.length(), mTextBound);
float textHeight = mTextBound.bottom - mTextBound.top;
```

然后设置画笔

```
mPaint.setStrokeWidth(0);
mPaint.setColor(mTextColor);
mPaint.setAntiAlias(true);
mPaint.setTextSize(mTextSize);
mPaint.setTypeface(Typeface.DEFAULT_BOLD);
Paint.FontMetrics fontMetrics = mPaint.getFontMetrics();
```

FontMetrics 类获取所画字符的位置属性，简单的说就是字体基于基准线的距离。

```
 /**
     * Class that describes the various metrics for a font at a given text size.
     * Remember, Y values increase going down, so those values will be positive,
     * and values that measure distances going up will be negative. This class
     * is returned by getFontMetrics().
     */
    public static class FontMetrics {
        /**
         * The maximum distance above the baseline for the tallest glyph in
         * the font at a given text size.
         */
        public float   top;
        /**
         * The recommended distance above the baseline for singled spaced text.
         */
        public float   ascent;
        /**
         * The recommended distance below the baseline for singled spaced text.
         */
        public float   descent;
        /**
         * The maximum distance below the baseline for the lowest glyph in
         * the font at a given text size.
         */
        public float   bottom;
        /**
         * The recommended additional space to add between lines of text.
         */
        public float   leading;
    }
```

然后就有了下面一些数据

```
D/( 7724): height : 40 textHeight :40.0
D/( 7724): width : 96 textWidth :102.0
D/( 7724): center : 120
D/( 7724):  top: -57.032227 ascent :-50.097656 descent :13.183594 bottom :14.633789 leading :0.0
```

借用其他人的图片
![](http://www.jcodecraeer.com/uploads/20130409/27381365503414.JPG)

有了这些数据我们就可以知道，如果通过FontMetrics会产生一些误差。

```
x = (view.width - text.width) /2;
y = (view.height)/2 - text.height/2;
```

这种情况下text是偏下的，由于没有精确的baseline，所以可以添加一个 调整值 beta

```
canvas.drawText(percent + "%", center - textWidth / 2, center + textHeight/2 , mPaint);
```

参考
[关于Android Canvas.drawText方法中的坐标参数的正确解释](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2013/0409/1143.html)
[Android Canvas drawText实现中文垂直居中](http://blog.csdn.net/hursing/article/details/18703599)

