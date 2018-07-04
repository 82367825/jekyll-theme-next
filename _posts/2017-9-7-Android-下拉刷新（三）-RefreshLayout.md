---
layout: post
title: Android-下拉刷新（三）-RefreshLayout
categories: [Android]
description: 
tags: 下拉刷新
---

时间推移，记得以前写过关于下拉刷新的博客文章

>Android-下拉刷新（一）http://blog.csdn.net/z82367825/article/details/52006656
Android-下拉刷新（二）http://blog.csdn.net/z82367825/article/details/52157358

现在回看代码，发现这种构建方式存在很大问题，当我项目中期突然需要一个ListView的带有下拉刷新功能，这时候，如果用以前的方式，就是用自定义的的RefreshListView去代替原生的ListView，这种体验非常一般，因为需要修改很多代码，而且，针对每一个类型的View，都要去实现一个自定义View。

Google官方对于下拉刷新自然早就有方案，在support library中加入了SwipeRefreshLayout布局，当开发者需要下拉刷新功能的时候，可以让SwipeRefreshLayout包裹任意目标视图，实现下拉效果。这时候我们真的体验到SwipeRefreshLayout的好处，不需要我们去修改自己的业务逻辑代码，就能轻松为应用加上刷新效果。于是，改进原来的体验较差的刷新库的想法涌上心头。

这篇博客主要是分析SwipeRefreshLayout原理，以及根据原理依样画葫芦，实现类似效果的下拉刷新布局。

# SwipeRefreshLayout 原理

首先我们看到SwipeRefreshLayout的简单实用：
```
<android.support.v4.widget.SwipeRefreshLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <ListView
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</android.support.v4.widget.SwipeRefreshLayout>
```

由于之前博客的经验，我们可以简单总结下拉刷新的关键步骤：
>1 对于触摸事件的拦截并处理
2 对于能否继续上滑或者下滑的判断


我们看到SwipeRefreshLayout#onInterceptTouchEvent()方法
```
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        ...
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                ...
                mIsBeingDragged = false;

                pointerIndex = ev.findPointerIndex(mActivePointerId);
                if (pointerIndex < 0) {
                    return false;
                }
                /* 记录手指按下的坐标Y值 */
                mInitialDownY = ev.getY(pointerIndex);
                break;

            case MotionEvent.ACTION_MOVE:
                ...
                /* 记录手指滑动的坐标Y值 */
                final float y = ev.getY(pointerIndex);
                /* 通过这里做判断是否处于下拉状态 */
                startDragging(y);
                break;
                ...
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                break;
        }

        return mIsBeingDragged;
    }
```
我们看到关键变量mIsBeingDragged，这个变量用来标记当前是否正处于下拉操作，如果当前正在下拉，则拦截触摸事件。这里主要是根据mIsBeingDragged的值，拦截触摸事件不让事件传递到子视图，直接交接触摸处理到onTouchEvent()方法里处理。

```
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ...
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mActivePointerId = ev.getPointerId(0);
                mIsBeingDragged = false;
                break;

            case MotionEvent.ACTION_MOVE: {
                ...
                final float y = ev.getY(pointerIndex);
                /* 通过这里做判断是否处于下拉状态 */
                startDragging(y);

                if (mIsBeingDragged) {
                    /* 计算手指滑动距离 */
                    final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
                    if (overscrollTop > 0) {
                        moveSpinner(overscrollTop);
                    } else {
                        return false;
                    }
                }
                break;
            }
            ...
            case MotionEvent.ACTION_UP: {
                ...
                if (mIsBeingDragged) {
                    final float y = ev.getY(pointerIndex);
                    final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
                    mIsBeingDragged = false;
                    finishSpinner(overscrollTop);
                }
                mActivePointerId = INVALID_POINTER;
                return false;
            }
            case MotionEvent.ACTION_CANCEL:
                return false;
        }

        return true;
    }

```
当触摸事件为MotionEvent.ACTION_MOVE的时候，此时开始滑动的时候，startDragging()方法被持续调用，当手指的滑动超过系统规定的滑动距离mTouchSlop，设置当前状态为下拉拖拽。

```
    @SuppressLint("NewApi")
    private void startDragging(float y) {
        final float yDiff = y - mInitialDownY;
        if (yDiff > mTouchSlop && !mIsBeingDragged) {
            mInitialMotionY = mInitialDownY + mTouchSlop;
            mIsBeingDragged = true;
            mProgress.setAlpha(STARTING_PROGRESS_ALPHA);
        }
    }
```
继续往下走，判断mIsBeingDragged，此时为滑动状态，计算当前滑动距离overscrollTop，如果滑动距离大于0，调用moveSpinner(overscrollTop)使得Spinner下拉并且旋转。
```
if (mIsBeingDragged) {
        final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
         if (overscrollTop > 0) {
            moveSpinner(overscrollTop);
         } else {
            return false;
         }
}
```
触摸事件的处理很简单，流程主要是先通过onInterceptTouchEvent拦截，再onTouchEvent做下拉处理。


接下来是对于能否继续上滑或者下滑的判断。
我们看到SwipeRefreshLayout#canChildScrollUp()代码

```
/**
     * @return Whether it is possible for the child view of this layout to
     *         scroll up. Override this if the child view is a custom view.
     */
    public boolean canChildScrollUp() {
        if (mChildScrollUpCallback != null) {
            return mChildScrollUpCallback.canChildScrollUp(this, mTarget);
        }
        if (android.os.Build.VERSION.SDK_INT < 14) {
            if (mTarget instanceof AbsListView) {
                final AbsListView absListView = (AbsListView) mTarget;
                return absListView.getChildCount() > 0
                        && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                                .getTop() < absListView.getPaddingTop());
            } else {
                return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
            }
        } else {
            return ViewCompat.canScrollVertically(mTarget, -1);
        }
    }
```
继续跟踪ViewCompat.canScrollVertically()

```
public static boolean canScrollVertically(View v, int direction) {
   return IMPL.canScrollVertically(v, direction);
}
```

继续跟踪，我们发现API小于14的时候调用的是BaseViewCompatImpl#canScrollVertically()（低版本这里就不讨论了），API大于等于14的时候，调用的是ICSViewCompatImpl#canScrollVertically()

看到ICSViewCompatImpl#canScrollVertically()

```
@Override
public boolean canScrollVertically(View v, int direction) {
    return ViewCompatICS.canScrollVertically(v, direction);
}
```
继续看ViewCompatICS#canScrollVertically
```
public static boolean canScrollVertically(View v, int direction) {
     return v.canScrollVertically(direction);
}

```
最后我们发现调用的是View#canScrollVertically()
```
    /**
     * Check if this view can be scrolled vertically in a certain direction.
     *
     * @param direction Negative to check scrolling up, positive to check scrolling down.
     * @return true if this view can be scrolled in the specified direction, false otherwise.
     */
    public boolean canScrollVertically(int direction) {
        final int offset = computeVerticalScrollOffset();
        final int range = computeVerticalScrollRange() - computeVerticalScrollExtent();
        if (range == 0) return false;
        if (direction < 0) {
            return offset > 0;
        } else {
            return offset < range - 1;
        }
    }
```
其实当API到大于等于14的时候，如果判断目标View是否可以继续下拉，都是通过来View#canScrollVertically()判断的，那么为什么通过这个方法就能实现对ListView, TextView, RecyclerView等View的适配呢？

我们发现computeVerticalScrollOffset(),computeVerticalScrollRange()这两个方法，基本上都被重写，通过重写这两个方法，实现了对视图是否可下拉的判断，详细的可以查看ListView或者RecyclerView等类的代码。

# 借鉴SwipeRefreshLayout实现下拉刷新库

基本了解了SwipeRefreshLayout的原理，我们可以模仿它，来实现一个带有下拉刷新以及上拉加载更多功能的布局。

例如如何实现是否可下拉或者是否可上拉的判断，这里我们可以直接借鉴SwipeRefreshLayout的代码
```
    /**
     * 是否可以继续下拉
     * @param targetView
     * @return
     */
    public static boolean canChildPullDown(View targetView) {
        /* 模仿SwipeRefreshLayout */
        if (android.os.Build.VERSION.SDK_INT < 14) {
            if (targetView instanceof AbsListView) {
                final AbsListView absListView = (AbsListView) targetView;
                return absListView.getChildCount() > 0
                        && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                        .getTop() < absListView.getPaddingTop());
            } else {
                return ViewCompat.canScrollVertically(targetView, -1) || targetView.getScrollY() > 0;
            }
        } else {
            return ViewCompat.canScrollVertically(targetView, -1);
        }
    }
```
同时上拉也一样，只需要把一些条件置反就行
```
    /**
     * 是否可以继续上拉
     * @param targetView
     * @return
     */
    public static boolean canChildPullUp(View targetView) {
        /* 模仿SwipeRefreshLayout */
        if (android.os.Build.VERSION.SDK_INT < 14) {
            if (targetView instanceof AbsListView) {
                final AbsListView absListView = (AbsListView) targetView;
                return absListView.getChildCount() > 0
                        && (absListView.getLastVisiblePosition() < absListView.getAdapter().getCount() - 1
                        || absListView.getChildAt(absListView.getChildCount() - 1).getBottom() > absListView.getPaddingBottom());
            } else {
                return ViewCompat.canScrollVertically(targetView, 1);
            }
        } else {
            return ViewCompat.canScrollVertically(targetView, 1);
        }
    }
```

其他具体的操作可以看项目源码。


# 下拉刷新库RefreshLayout介绍

github地址：https://github.com/82367825/RefreshLayout

## 如何引入

在build.gradle文件中加入
```
compile 'com.zero.refreshlayout.library:RefreshLayout:1.0.0'
```

## 简单使用

首先是xml代码：

```
    <com.zero.refreshlayout.library.RefreshLayout
        android:id="@+id/refresh_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <TextView
            android:id="@+id/text_view"
            android:text="text"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
        
    </com.zero.refreshlayout.library.RefreshLayout>
```

在java代码设置相关的属性，并且设置监听器

```
mRefreshLayout = (RefreshLayout) findViewById(R.id.refresh_layout);
mRefreshLayout.setHeaderView(new TextHeaderView(this));
mRefreshLayout.setFooterView(new TextFooterView(this));
mRefreshLayout.setRefreshListener(new RefreshListener() {
            @Override
            public void onRefresh() {
                MainThreadHandler.execute(new Runnable() {
                    @Override
                    public void run() {
                        mRefreshLayout.finishRefresh();
                    }
                }, 2000);
            }

            @Override
            public void onLoadMore() {
                MainThreadHandler.execute(new Runnable() {
                    @Override
                    public void run() {
                        mRefreshLayout.finishLoadMore();
                    }
                }, 2000);
            }
        });
```

更多的开放API：

>* void finishRefresh()
结束下拉刷新状态
>* void finishLoadMore()
结束上拉加载状态
>* void setHeaderViewHeight(int headerViewHeight)
设置HeaderView的高度,如果不设置的话，HeaderView默认为wrap_content
>* void setFooterViewHeight(int footerViewHeight)
设置FooterView的高度,如果不设置的话，FooterView默认为wrap_content
>* setHeaderViewMaxPullDistance(int headerViewMaxPullDistance)
设置下拉最大拉伸长度，如果不设置的话，默认为HeaderView高度的2.5倍
>* setFooterViewMaxPullDistance(int footerViewMaxPullDistance)
设置上拉最大拉伸长度，如果不设置的话，默认为FooterView高度的2.5倍
>* void setHeaderViewEnable(boolean headerViewEnable)
设置HeaderView是否可用
>* void setFooterViewEnable(boolean footerViewEnable) 
设置FooterView是否可用


## 个性化使用

### 1）布局样式多样化

如果你对视图的布局样式不喜欢，这里暂时提供了另外两种布局样式。

如何切换布局样式，只需要调用
```
void setRefreshMode(RefreshMode refreshMode)
```

#### 默认样式 RefreshMode.LINEAR 

```
mRefreshLayout.setRefreshMode(RefreshMode.LINEAR);
```

![这里写图片描述](http://img.blog.csdn.net/20170907163209988?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


#### 覆盖样式 RefreshMode.COVER_LAYOUT_FRONT (布局覆盖在下拉控件之前)

```
mRefreshLayout.setRefreshMode(RefreshMode.COVER_LAYOUT_FRONT);
```
![这里写图片描述](http://img.blog.csdn.net/20170907163451311?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 覆盖样式 RefreshMode.COVER_LAYOUT_BEHIND (布局被下拉控件覆盖)

```
mRefreshLayout.setRefreshMode(RefreshMode.COVER_LAYOUT_BEHIND);
```

![这里写图片描述](http://img.blog.csdn.net/20170907163520371?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 2) 定制HeaderView以及FooterView

最重要当然是自己定制HeaderView以及FooterView，通过下面简单的例子我们就能轻松看懂

所有的HeaderView都必须继承AbsHeaderView, 所有的FooterView都必须继承AbsFooterView，我们看到，这里实现了一个TextHeaderView类

```
public class TextHeaderView extends AbsHeaderView {

    private TextView mTextView;
    
    public TextHeaderView(Context context) {
        super(context);
        mTextView = new TextView(context);
        mTextView.setBackgroundColor(0xFFFFFFFF);
        mTextView.setTextColor(0xFF000000);
        mTextView.setTextSize(20);
        mTextView.setPadding(50, 50, 50, 50);
        mTextView.setGravity(Gravity.CENTER);
        mTextView.setText("下拉刷新");
    }

    @Override
    public View getContentView() {
        return mTextView;
    }

    @Override
    public void onPullDown(float fraction) {
        mTextView.setText("下拉刷新");
    }

    @Override
    public void onFinishPullDown(float fraction) {
        
    }

    @Override
    public void onRefreshing() {
        mTextView.setText("刷新...");
    }
}

```

>* AbsHeaderView#onPullDown
   HeaderView正在被下拉时调用，fraction参数为HeaderView完全显示的比例
>* AbsHeaderView#onFinishPullDown
   HeaderView被松手弹回，fraction参数为HeaderView完全显示的比例
>* AbsHeaderView#onRefreshing
   HeaderView在处于刷新状态
   


### 3）默认提供的HeaderView，FooterView主题

如果我们只是想快速使用，可以使用现成的HeaderView以及FooterView, RefreshLayout暂时提供了几个样式

```
mRefreshLayout.setHeaderView(new BezierWaveHeader(this));
mRefreshLayout.setFooterView(new BezierWaveFooter(this));
```
![这里写图片描述](http://img.blog.csdn.net/20170907163608030?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```
mRefreshLayout.setHeaderView(new SquareSpreadHeader(this));
mRefreshLayout.setFooterView(new SquareSpreadFooter(this));
```
![这里写图片描述](http://img.blog.csdn.net/20170907163757753?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 更新历史

>*  v1.0.1  
修复bug


学习不易，可能会有很多问题，欢迎拍砖

github地址：https://github.com/82367825/RefreshLayout
