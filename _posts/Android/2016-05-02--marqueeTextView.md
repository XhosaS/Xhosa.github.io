---
layout: post
title: Android TextView实现跑马灯特效
category: Android
tags: Andorid
description: Android TextView实现跑马灯特效
---

## 一般情况
直接在.xml 布局文件中，`TextView`的属性定义添加

      android:singleLine="true"
      android:ellipsize="marquee"
      android:marqueeRepeatLimit="marquee_forever"
      android:scrollHorizontally="true"

这样就可以使得TextView进行跑马灯特效了，但是前期是这个TextView必须获取焦点，这是android源码规定的，如果yao要不获取焦点也能滚动，那么需要一些改动。

## 不获取焦点也能实现跑马灯特效

创建一个AlwaysMarqueeTextView，继承`TextView`，代码如下:

    package com.xhosa.xhosamusic;

    import android.widget.TextView;
    import android.content.Context;
    import android.util.AttributeSet;
    import android.widget.RemoteViews.RemoteView;
    /**
    * Created by 刘润锦 on 2016/3/3.
    */
    public class AlwaysMarqueeTextView extends TextView {

        public AlwaysMarqueeTextView(Context context) {
            super(context);
        }
        public AlwaysMarqueeTextView(Context context, AttributeSet attrs) {
            super(context, attrs);
        }
        public AlwaysMarqueeTextView(Context context, AttributeSet attrs,
                                     int defStyle) {
            super(context, attrs, defStyle);
        }
        @Override
        public boolean isFocused() {
            return true;
        }


    }

主要是重写了isFocused()方法，让isFocused()始终返回true，从而让TextView实现默认跑马灯效果。然后再在.xml文件中用`AlwaysMarqueeTextView`替代TextView，就可以了。

## 其他
在本方法中，重写了TextView的isFocused()，但我尚不清楚这一改动会不会造成其他影响，如果有，将在这里进行更新。
