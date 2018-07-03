---
layout: post
title: Android-ViewPager嵌套使用探究
categories: [Android]
description: ViewPager的嵌套使用时非常频繁的,我们如何对它们做优化处理呢
tags: ViewPager 
---
>终于不再是实习生，走出了穷逼的实习生生活，正式开始稍微不那么穷逼的正职员工生活，三天的启航培训还是挺欢乐的，学到一句话，保持饥渴，哈哈，感觉放到哪方面都说得通。

## 关于ViewPager嵌套

ViewPager的嵌套使用是一个很常见的问题，然而，最近又一次遇到ViewPager的嵌套使用问题。

情景是这样的，需求上给出了这样的要求，需要实现内外两个ViewPager嵌套的效果，外部ViewPager控制着4个Tab的切换滑动，内部ViewPager控制着若干个二级Tab的滑动切换（这里可能是广告栏，也可能是榜单等），另外，当内部ViewPager滑动到最左或者最右的时候，外部ViewPager恢复滑动。

需求刚一看的时候，确实不难，于是我立马想出了方案一。

## 方案一

解决两个ViewPager滑动冲突问题，首先我们要知道为什么会产生滑动冲突，通过查看源码我们大致发现，是由于ViewPager的onInteruptTouchEvent()方法拦截了传递给子View的触摸事件。

那么从这个点出发，解决方法就是，重写内部ViewPager，调用getParent().requestDisallowInterceptTouchEvent(true)方法来禁止外部ViewPager拦截触摸事件，使得内部ViewPager可以滑动。

代码：

```java
package com.zero.viewpagerdemo.way1;

import android.content.Context;
import android.support.v4.view.ViewPager;
import android.util.AttributeSet;
import android.view.MotionEvent;

/**
 * @author linzewu
 * @date 16-7-12
 */
public class InnerViewPager extends ViewPager {


    public InnerViewPager(Context context) {
        super(context);
    }

    public InnerViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }


    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
}


```

这个方案实现了两个ViewPager嵌套时，保证了内部ViewPager能够正常滑动，同时外部ViewPager可以滑动，但是，当手指触摸范围在内部ViewPager内的时候，而且内部ViewPager滑动了最左或者最右边的时候，理想状态下是此时外部ViewPager恢复响应，可以滑动，如果按照以上代码，就实现不了这一步。

## 方案二

既然我们要监控内部ViewPager是否滑动最左或者最右再来判断是否让触摸事件传递给内部ViewPager，显然从内部ViewPager去实现似乎不太理想，那就从外部ViewPager着手处理，
我们去试着重写外部ViewPager.

于是，我想了一个能够解决问题，但不是很高级的方法.
让外部ViewPager持有内部ViewPager的引用，这样外部ViewPager就可以在onInterceptTouchEvent()方法里通过判断内部ViewPager是否滑动到了最左或者最右。

```java
package com.zero.viewpagerdemo.way2;

import android.content.Context;
import android.support.v4.view.ViewPager;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;

import java.util.List;


/**
 * 嵌套ViewPager外部ViewPager滑动冲突解决
 * 
 * @author linzewu
 * @date 16-7-12
 */
public class OuterViewPager extends ViewPager {
    public OuterViewPager(Context context) {
        super(context);
    }

    public OuterViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    
    private float mDownX;
    private float mMoveX;

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mDownX = ev.getRawX();
                break;
            case MotionEvent.ACTION_MOVE:
                mMoveX = ev.getRawX();
                InnerViewPager currentViewPager = getCurrentInnerViewPager();
                if (currentViewPager != null) {
                    if (mMoveX - mDownX > 0 && !isViewPagerReachLeft(getCurrentInnerViewPager()
                            .mViewPager)) {
                        return false;
                    } else if (mMoveX - mDownX <= 0 && !isViewPagerReachRight
                            (getCurrentInnerViewPager().mViewPager)) {
                        return false;
                    }
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                break;
        }
        return super.onInterceptTouchEvent(ev);
    }
    

    /**
     * 内部ViewPager集合
     */
    private List<InnerViewPager> mInnerViewPagers;
    
    public void setInnerViewPagers(List<InnerViewPager> innerViewPagers) {
        this.mInnerViewPagers = innerViewPagers;
    }

    /**
     * 内部ViewPager类
     */
    public static class InnerViewPager {
        ViewPager mViewPager;
        int mIndex;  //内部viewPager在外部ViewPager的位置值
        public InnerViewPager(ViewPager viewPager, int index) {
            this.mViewPager = viewPager;
            this.mIndex = index;
        }
        
    }
    
    private InnerViewPager getCurrentInnerViewPager() {
        if (mInnerViewPagers == null) {
            return null;
        }
        for (InnerViewPager innerViewPager : mInnerViewPagers) {
            if (innerViewPager.mIndex == getCurrentItem()) {
                return innerViewPager;
            }
        }
        return null;
    }
    
    private boolean isViewPagerReachLeft(ViewPager viewPager) {
        return viewPager.getCurrentItem() == 0;
    }
    
    private boolean isViewPagerReachRight(ViewPager viewPager) {
        return viewPager.getCurrentItem() >= viewPager.getChildCount() - 1
                && viewPager.getChildCount() == 2;
    }
}


```

采用重写外部ViewPager的方式，由于需求需要，当内部ViewPager滑动到最左或者最右的时候，外部ViewPager能够恢复响应，缺点：需要保留内部ViewPager的引用，导致该控件拓展性不高，这是一种很不高级的重写方式。

## 方案三

通过以上方案，其实已经是可以解决ViewPager的滑动问题，但是有另一个问题一直困扰着我.因为一开始使用ViewPager嵌套出现滑动冲突的情况，只是在2.3的机器上出现，而在高版本的机器例如5.0，使用ViewPager嵌套ViewPager，是不会出现滑动冲突的情况，这就相当奇怪了，既然高版本不会出现滑动冲突，为什么在低版本上反而会出现滑动冲突呢？

带着这些疑问，我开始去翻查资料.


我们先找到ViewPager的onInterceptTouchEvent()方法

```java
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
    
    ...//省略不展示
    
    switch (action) {
            case MotionEvent.ACTION_MOVE: {
                /*
                 * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
                 * whether the user has moved far enough from his original down touch.
                 */

                /*
                * Locally do absolute value. mLastMotionY is set to the y value
                * of the down event.
                */
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }

                final int pointerIndex = MotionEventCompat.findPointerIndex(ev, activePointerId);
                final float x = MotionEventCompat.getX(ev, pointerIndex);
                final float dx = x - mLastMotionX;
                final float xDiff = Math.abs(dx);
                final float y = MotionEventCompat.getY(ev, pointerIndex);
                final float yDiff = Math.abs(y - mInitialMotionY);
                if (DEBUG) Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);

                if (dx != 0 && !isGutterDrag(mLastMotionX, dx) &&
                        canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    // Nested view has scrollable area under this point. Let it be handled there.
                    mLastMotionX = x;
                    mLastMotionY = y;
                    mIsUnableToDrag = true;
                    return false;
                }
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0 ? mInitialMotionX + mTouchSlop :
                            mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {
                    // The finger has moved enough in the vertical
                    // direction to be counted as a drag...  abort
                    // any attempt to drag horizontally, to work correctly
                    // with children that have scrolling containers.
                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");
                    mIsUnableToDrag = true;
                }
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    if (performDrag(x)) {
                        ViewCompat.postInvalidateOnAnimation(this);
                    }
                }
                break;
            }
     
     ...
     }       
            

```
从上面代码我们可以看出，当判断为ACTION_MOVE的时候，这里首先会判断dx不为0，如果为，0，同时再判断canScroll()方法返回是否为true,如果为true，则onInterceptTouchEvent整个方法体返回false,也就是当前ViewPager不拦截处理触摸滑动事件，那么这些一连串的触摸滑动事件则会传递给子ViewPager处理。


再来看这个关键方法canScroll()，从官方的注释，我们可以知道这个方法的作用是测试某个View的可滑动性.参数checkV很重要，当checkV为ture,返回的是View本身加上子View的可滑动性，当checkV为false,返回的只是View的子View的可滑动性。

我们可以看到代码中先判断v是否为ViewGroup，如果为ViewGroup,则依次递归调用canScroll()来判断子View的可滑动性，注意这里canScroll()的checkV的值为true.最后一行则是判断自己View本身的可滑动性，当然当checkV为false的时候，不会判断自己的可滑动性。

```java
    /**
     * Tests scrollability within child views of v given a delta of dx.
     *
     * @param v View to test for horizontal scrollability
     * @param checkV Whether the view v passed should itself be checked for scrollability (true),
     *               or just its children (false).
     * @param dx Delta scrolled in pixels
     * @param x X coordinate of the active touch point
     * @param y Y coordinate of the active touch point
     * @return true if child views of v can be scrolled by delta of dx.
     */
    protected boolean canScroll(View v, boolean checkV, int dx, int x, int y) {
        if (v instanceof ViewGroup) {
            final ViewGroup group = (ViewGroup) v;
            final int scrollX = v.getScrollX();
            final int scrollY = v.getScrollY();
            final int count = group.getChildCount();
            // Count backwards - let topmost views consume scroll distance first.
            for (int i = count - 1; i >= 0; i--) {
                // TODO: Add versioned support here for transformed views.
                // This will not work for transformed views in Honeycomb+
                final View child = group.getChildAt(i);
                if (x + scrollX >= child.getLeft() && x + scrollX < child.getRight() &&
                        y + scrollY >= child.getTop() && y + scrollY < child.getBottom() &&
                        canScroll(child, true, dx, x + scrollX - child.getLeft(),
                                y + scrollY - child.getTop())) {
                    return true;
                }
            }
        }

        return checkV && ViewCompat.canScrollHorizontally(v, -dx);
    }
```

而在上文中，我们知道ViewPager在onInterceptTouchEvent()方法中调用的代码是

```
canScroll(this, false, (int) dx, (int) x, (int) y)
```
checkV的值为false,因此返回的是ViewPager的子View的可滑动性。






再进一步查看ViewCompat.canScrollHorizontally(）的源码

```java

    /**
     * Check if this view can be scrolled horizontally in a certain direction.
     *
     * @param v The View against which to invoke the method.
     * @param direction Negative to check scrolling left, positive to check scrolling right.
     * @return true if this view can be scrolled in the specified direction, false otherwise.
     */
    public static boolean canScrollHorizontally(View v, int direction) {
        return IMPL.canScrollHorizontally(v, direction);
    }
    
```

canScrollHorizontally()用来判断当前View的水平滑动性，也就是这个是否还可以滑动。我们发现，这个方法在不同的版本，有不同的方法体。

```java
    static final ViewCompatImpl IMPL;
    static {
        final int version = android.os.Build.VERSION.SDK_INT;
        if (version >= 23) {
            IMPL = new MarshmallowViewCompatImpl();
        } else if (version >= 21) {
            IMPL = new LollipopViewCompatImpl();
        } else if (version >= 19) {
            IMPL = new KitKatViewCompatImpl();
        } else if (version >= 17) {
            IMPL = new JbMr1ViewCompatImpl();
        } else if (version >= 16) {
            IMPL = new JBViewCompatImpl();
        } else if (version >= 15) {
            IMPL = new ICSMr1ViewCompatImpl();
        } else if (version >= 14) {
            IMPL = new ICSViewCompatImpl();
        } else if (version >= 11) {
            IMPL = new HCViewCompatImpl();
        } else if (version >= 9) {
            IMPL = new GBViewCompatImpl();
        } else if (version >= 7) {
            IMPL = new EclairMr1ViewCompatImpl();
        } else {
            IMPL = new BaseViewCompatImpl();
        }
    }
```

当API 小于7 或者 小于9 或者 小于11 或者 小于14的时候，也就是IMPL的实例为BaseViewCompatImpl，GBViewCompatImpl, EclairMr1ViewCompatImpl，HCViewCompatImpl，它们的canScrollHorizontally方法体是一样的，因为基础类都是BaseViewCompatImpl.

```java
public boolean canScrollHorizontally(View v, int direction) {
      return (v instanceof ScrollingView) &&
                canScrollingViewScrollHorizontally((ScrollingView) v, direction); 
  }
```

只有当View实现ScrollingView这个接口的时候，才不会返回false，但是我们都知道ViewPager是继承ViewGroup而来，也没有实现这个接口，因此这里肯定是返回false了。

然而当API大于等于14，IMPL的实例为ICSViewCompatImpl，这时候复写了canScrollHorizontally()方法。

```java
@Override
public boolean canScrollHorizontally(View v, int direction) {
      return ViewCompatICS.canScrollHorizontally(v, direction);
}
```
我们继续跟踪进入，发现最后调用的是View.canScrollHorizontally()方法。

```java
    /**
     * Check if this view can be scrolled horizontally in a certain direction.
     *
     * @param direction Negative to check scrolling left, positive to check scrolling right.
     * @return true if this view can be scrolled in the specified direction, false otherwise.
     */
    public boolean canScrollHorizontally(int direction) {
        final int offset = computeHorizontalScrollOffset();
        final int range = computeHorizontalScrollRange() - computeHorizontalScrollExtent();
        if (range == 0) return false;
        if (direction < 0) {
            return offset > 0;
        } else {
            return offset < range - 1;
        }
    }
```

这样一来，我们可以做出总结了，当API小于14的时候，ViewPager的canScroll()方法无法获知子View的可滑动性（因为它只会默认返回false，总是告诉你子View不可滑动），当API大于等于14的时候，ViewPager的canScroll()方法能够最终调用到View.canScrollHorizontally(）,即能够获知子View的可滑动性。


同时，我们也知道了，为什么在2.3的机型上，使用ViewPager嵌套ViewPager会出现滑动冲突，而在高版本上，这种让人郁闷的现象却神奇地恢复了。


那么，如果解决这个问题呢，既然canScroll()方法中，当API小于14的时候，默认返回false，我们可以试着让它最终去调用View.canScrollHorizontally（)，实现和当API大于等于14一样的效果。

```java
package com.zero.viewpagerdemo.way3;

import android.content.Context;
import android.support.v4.view.ViewCompat;
import android.support.v4.view.ViewPager;
import android.util.AttributeSet;
import android.view.View;
import android.view.ViewGroup;

/**
 * @author linzewu
 * @date 16-7-12
 */
public class CompatibleViewPager extends ViewPager {
    public CompatibleViewPager(Context context) {
        super(context);
    }

    public CompatibleViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected boolean canScroll(View v, boolean checkV, int dx, int x, int y) {

        if (v instanceof ViewGroup) {
            final ViewGroup group = (ViewGroup) v;
            final int scrollX = v.getScrollX();
            final int scrollY = v.getScrollY();
            final int count = group.getChildCount();
            // Count backwards - let topmost views consume scroll distance first.
            for (int i = count - 1; i >= 0; i--) {
                // TODO: Add versioned support here for transformed views.
                // This will not work for transformed views in Honeycomb+
                final View child = group.getChildAt(i);
                if (x + scrollX >= child.getLeft() && x + scrollX < child.getRight() &&
                        y + scrollY >= child.getTop() && y + scrollY < child.getBottom() &&
                        canScroll(child, true, dx, x + scrollX - child.getLeft(),
                                y + scrollY - child.getTop())) {
                    return true;
                }
            }
        }
        
        if (checkV) {
            if (v instanceof ViewPager) {
                return ((ViewPager)v).canScrollHorizontally(-dx);
            } else {
                return ViewCompat.canScrollHorizontally(v, -dx);
            }
        } else {
            return false;
        }
    }
}

```

最后，通过在demo的试验，发现通过这种方式去重写ViewPager，确实可以解决ViewPager嵌套使用的滑动冲突问题，而且暂时没有发现一些其他的问题。

Demo源码：[传送门](http://download.csdn.net/detail/z82367825/9578030 "Title")  

如有问题，欢迎指正 .





参考：http://www.myexception.cn/android/1901549.html



