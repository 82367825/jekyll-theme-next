---
layout: post
title: Android-ValueAnimator内存泄露
categories: [Android]
description: 
tags: 内存泄露
---


今天检查代码的时候发现了一个内存泄漏的问题，导致Activity内存一直无法释放，后来发现是Activity内部的全局变量mValueAnimator无法释放而导致的。

### 代码分析

我们先看到代码，为了实现一个动画效果，我们在Activity内放置了一个ValueAnimator的全局变量，并且调用开启动画的方法。

```
 private ValueAnimator mValueAnimator;
 private void initAnimation() {
        mValueAnimator = ValueAnimator.ofFloat(0, 1f);
        mValueAnimator.setDuration(500);
        mValueAnimator.setInterpolator(new CycleInterpolator(1));
        mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
            }
        });
        mValueAnimator.setRepeatCount(ValueAnimator.INFINITE);
        mValueAnimator.setRepeatMode(ValueAnimator.RESTART);
        mValueAnimator.start();
    }
```
然而，在没有做任何操作之后退出了Activity页面，这时候出现了内存泄漏，通过分析Java Head内存，我们发现是mValueAnimator对象没有得到释放导致。

于是我们在Activity#onDestory()方法中添加了代码

```
if (mValueAnimator != null) {
    mValueAnimator.cancel();
    mValueAnimator = null;
}
```
结果和预期一样，内存泄漏不再出现了。

那么为什么不对ValueAnimator动画做取消动作，就会出现内存泄漏呢？

我们看到ValueAnimator源码就能得到答案。

我们直接看到ValueAnimator#cancel()方法

```
@Override
    public void cancel() {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        if (mAnimationEndRequested) {
            return;
        }
        if ((mStarted || mRunning) && mListeners != null) {
            if (!mRunning) {
                notifyStartListeners();
            }
            ArrayList<AnimatorListener> tmpListeners =
                    (ArrayList<AnimatorListener>) mListeners.clone();
            for (AnimatorListener listener : tmpListeners) {
                listener.onAnimationCancel(this);
            }
        }
        endAnimation();
    }
```

基本看起来，都没有什么释放引用的操作，那么直接看到最后一行，继续查看endAnimation()方法。

```
private void endAnimation() {
        if (mAnimationEndRequested) {
            return;
        }
        AnimationHandler handler = AnimationHandler.getInstance();
        handler.removeCallback(this);
		 ...
    }
```

好了，我们已经找到了，AnimationHandler实现了单例模式，这里AnimationHandler移除了ValueAnimator的动画监听回调。如果我们不调用ValueAnimator#cancel方法，使得AnimationHandler这个单例对象释放ValueAnimator的引用，ValueAnimator对象的内存就会得不到释放，也就造成了内存泄漏。




因此，很简单，我们在使用动画类的时候，要注意在生命周期结束的时候，取消或者停止动画，以防出现内存泄漏问题。
