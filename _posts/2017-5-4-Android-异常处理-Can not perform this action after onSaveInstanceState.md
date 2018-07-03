---
layout: post
title: Android-异常处理-Can not perform this action after onSaveInstanceState
categories: [Android]
description: Android异常处理
tags: 异常
---

今天在奔溃日志上看到这样的错误信息：

```
Can not perform this action after onSaveInstanceStatejava.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
at android.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1441)
at android.app.FragmentManagerImpl.popBackStackImmediate(FragmentManager.java:581)
at android.app.Activity.onBackPressed(Activity.java:2542)
```

从错误信息我们可以看到，问题所在是Activity#onBackPressed()方法

我们看源码

Activity#onBackPressed()

```
    /**
     * Called when the activity has detected the user's press of the back
     * key.  The default implementation simply finishes the current activity,
     * but you can override this to do whatever you want.
     */
    public void onBackPressed() {
        if (mActionBar != null && mActionBar.collapseActionView()) {
            return;
        }

        if (!mFragments.getFragmentManager().popBackStackImmediate()) {
            finishAfterTransition();
        }
    }
```

我们看到onBackPressed()方法执行了两个操作，第一个是获取当前的FragmentManager，并且执行退栈操作，第二个是在退栈完成之后，执行finish方法。

继续查看源码，关键是FragmentManager实现类的popBackStackImmediate方法

```
    @Override
    public boolean popBackStackImmediate() {
        checkStateLoss();
        return popBackStackImmediate(null, -1, 0);
    }
```

我们看到，在执行退栈动作之前，这里还有一步检查操作

```
  private void checkStateLoss() {
        if (mStateSaved) {
            throw new IllegalStateException(
                    "Can not perform this action after onSaveInstanceState");
        }
        if (mNoTransactionsBecause != null) {
            throw new IllegalStateException(
                    "Can not perform this action inside of " + mNoTransactionsBecause);
        }
    }
```

从这里，我们终于找到了崩溃日志上的异常文案：

```
Can not perform this action after onSaveInstanceState

```

那么什么时候mStateSaved的值会是true？当Activity被销毁之前，系统会调用onSaveInstanceState来保存当前页面数据，当这个方法被执行之后，mSateSaved会被置为true.



## 分析

通过上面的源码分析，我们可以知道，出现以上崩溃日志的原因，是因为我们在按下页面返回键的时候，当前Activity以及在执行销毁操作（也就是说我们以前在其他地方调用了finish方法）。


## 如何解决

方案1 在调用super.onBackPressed的时候，我们需要判断当前Activity是否正在执行销毁操作。

```
if (!isFinishing()) {
	super.onBackPressed();
}
```

方案2 通过上面的源码分析，我们也知道了，super.onBackPressed最后也是调用finish()方法，因此我们可以重写onBackPressed，直接调用finish方法。
