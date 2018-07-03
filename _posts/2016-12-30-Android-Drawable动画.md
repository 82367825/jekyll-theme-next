---
layout: post
title: Android-Drawable动画
categories: [Android]
description: 自定义Drawable实现动画效果远比我们想象得要简单快捷得多
tags: Drawable 
---
# 前言
之前实现自定义动画或者视图都是通过自定义View来实现，通过重写onMeasure()，onLayout()，onDraw()等方法，然而其实我们也可以通过自定义Drawable来实现自定义动画的效果。自定义Drawable相对来说更加简洁，我们实际上只需要关注draw()方法。

# 关于Drawable
Drawable是一个表示我们想要绘制的事物的抽象体，不像View视图，Drawable并没有事件接收机制体系，也会大小测量等方法，它就是单纯的一个用来表示绘制视图的类，为了方便简单的绘制，Drawable为我们提供了几个辅助绘制的方法：
```java
###setBounds
The method must be called to tell the Drawable where it is drawn and how large it should be. All Drawables should respect the requested size, often simply by scaling their imagery.  A client can find the preferred size for some Drawables with the {@link #getIntrinsicHeight} and {@link #getIntrinsicWidth} methods.

###setState
The method allows the client to tell the Drawable in which state it is to be drawn, such as "focused", "selected", etc.Some drawables may modify their imagery based on the selected state.

###setLevel
The method allows the client to supply a single continuous controller that can modify the Drawable is displayed, such as a battery level or progress level.  Some drawables may modify their imagery based on the current level.

###Callback
A Drawable can perform animations by calling back to its client through the {@link Callback} interface.  All clients should support this interface (via {@link #setCallback}) so that animations will work.  A simple way to do this is through the system facilities such as {@link android.view.View#setBackground(Drawable)} and {@link android.widget.ImageView}.
```

## ImageView处理Drawable分析

我们都知道ImageView可以设置Drawable作为其视图，这个Drawable一般是BitmapDrawable，我们来看看这个setImageDrawable()方法体里是如何实现的。

ImageView#setImageDrawable()
```java
    /**
     * Sets a drawable as the content of this ImageView.
     * 
     * @param drawable the Drawable to set, or {@code null} to clear the
     *                 content
     */
    public void setImageDrawable(@Nullable Drawable drawable) {
        if (mDrawable != drawable) {
            mResource = 0;
            mUri = null;

            final int oldWidth = mDrawableWidth;
            final int oldHeight = mDrawableHeight;
            //更新Drawable
            updateDrawable(drawable);

            if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                requestLayout();
            }
            invalidate();
        }
    }
```

继续跟踪到updateDrawable()方法体里面，我们可以看到通过给mDrawableWidth和mDrawableHeight赋值，获取了Drawable的内部宽度和内部高度。

ImageView#updateDrawable()
```java
    private void updateDrawable(Drawable d) {
        if (d != mRecycleableBitmapDrawable && mRecycleableBitmapDrawable != null) {
            mRecycleableBitmapDrawable.setBitmap(null);
        }

        if (mDrawable != null) {
            mDrawable.setCallback(null);
            unscheduleDrawable(mDrawable);
        }

        mDrawable = d;

        if (d != null) {
            d.setCallback(this);
            d.setLayoutDirection(getLayoutDirection());
            if (d.isStateful()) {
                d.setState(getDrawableState());
            }
            d.setVisible(getVisibility() == VISIBLE, true);
            d.setLevel(mLevel);
            //获取Drawable的内部宽度和内部高度
            mDrawableWidth = d.getIntrinsicWidth();
            mDrawableHeight = d.getIntrinsicHeight();
            applyImageTint();
            applyColorMod();

            configureBounds();
        } else {
            mDrawableWidth = mDrawableHeight = -1;
        }
    }
```

上面保存了Drawable的内部宽度和内部高度，那么这里两个值有什么用呢，我们继续看到onMeasure()方法体里。

ImageView#onMeasure()
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        int w;
        int h;
        ...
        if (mDrawable == null) {
            // If no drawable, its intrinsic size is 0.
            mDrawableWidth = -1;
            mDrawableHeight = -1;
            w = h = 0;
        } else {
            w = mDrawableWidth;
            h = mDrawableHeight;
            if (w <= 0) w = 1;
            if (h <= 0) h = 1;

            // We are supposed to adjust view bounds to match the aspect
            // ratio of our drawable. See if that is possible.
            // mAdjustViewBounds参数，当它的值为true，表示ImageView会调整宽高来保证图像的纵横比，但是当View的尺寸模式是EXACTLY的时候，这种调整是无效的，因为宽度高度都固定。
            if (mAdjustViewBounds) {
                resizeWidth = widthSpecMode != MeasureSpec.EXACTLY;
                resizeHeight = heightSpecMode != MeasureSpec.EXACTLY;
                
                desiredAspect = (float) w / (float) h;
            }
        }
        
    
    }
```

从上面可以知道，getIntrinsicWidth()方法以及getIntrinsicHeight()在继承Drawable类的子类中是很重要的，它被用来声明Drawable的实际宽度和高度，当Drawable作为ImageView的资源展示的时候，如果尺寸为wrap_content，那么这时候ImageView的宽度和高度会设置为getIntrinsicWidth()和getIntrinsicHeight()，如果尺寸为确定值，比如xxdp或者match_parent，ImageView显示的视图会按照比例缩放。

```java
    /**
     * Return the intrinsic width of the underlying drawable object.  Returns
     * -1 if it has no intrinsic width, such as with a solid color.
     */
    public int getIntrinsicWidth() {
        return -1;
    }

    /**
     * Return the intrinsic height of the underlying drawable object. Returns
     * -1 if it has no intrinsic height, such as with a solid color.
     */
    public int getIntrinsicHeight() {
        return -1;
    }
```



分析另一个现象，当我们为大小为100*100的ImageView设置一个内部尺寸为60*60的Drawable时，我们看到的显示的Drawable不是60*60，而是和ImageView一样大小。

接下来我们看到一个方法体configureBounds(),通过上面的代码分析我们看到，在setImageDrawable()方法体中我们为ImageView设置Drawable，然后又调用了updateDrawable()，又再调用了configureBounds()，通过方法体名字，我们可以大概猜出，这个方法是用来设置Drawable的显示尺寸的。

ImageView#configureBounds()
```java
   private void configureBounds() {
        if (mDrawable == null || !mHaveFrame) {
            return;
        }

        int dwidth = mDrawableWidth;
        int dheight = mDrawableHeight;

        int vwidth = getWidth() - mPaddingLeft - mPaddingRight;
        int vheight = getHeight() - mPaddingTop - mPaddingBottom;

        boolean fits = (dwidth < 0 || vwidth == dwidth) &&
                       (dheight < 0 || vheight == dheight);

        if (dwidth <= 0 || dheight <= 0 || ScaleType.FIT_XY == mScaleType) {
            /* If the drawable has no intrinsic size, or we're told to
                scaletofit, then we just fill our entire view.
            */
            mDrawable.setBounds(0, 0, vwidth, vheight);
            mDrawMatrix = null;
        } else {
            // We need to do the scaling ourself, so have the drawable
            // use its native size.
            mDrawable.setBounds(0, 0, dwidth, dheight);

            if (ScaleType.MATRIX == mScaleType) {
                // Use the specified matrix as-is.
                if (mMatrix.isIdentity()) {
                    mDrawMatrix = null;
                } else {
                    mDrawMatrix = mMatrix;
                }
            } else if (fits) {
                // The bitmap fits exactly, no transform needed.
                mDrawMatrix = null;
            } else if (ScaleType.CENTER == mScaleType) {
                // Center bitmap in view, no scaling.
                mDrawMatrix = mMatrix;
                mDrawMatrix.setTranslate(Math.round((vwidth - dwidth) * 0.5f),
                                         Math.round((vheight - dheight) * 0.5f));
            } else if (ScaleType.CENTER_CROP == mScaleType) {
                mDrawMatrix = mMatrix;

                float scale;
                float dx = 0, dy = 0;

                if (dwidth * vheight > vwidth * dheight) {
                    scale = (float) vheight / (float) dheight; 
                    dx = (vwidth - dwidth * scale) * 0.5f;
                } else {
                    scale = (float) vwidth / (float) dwidth;
                    dy = (vheight - dheight * scale) * 0.5f;
                }

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(Math.round(dx), Math.round(dy));
            } else if (ScaleType.CENTER_INSIDE == mScaleType) {
                mDrawMatrix = mMatrix;
                float scale;
                float dx;
                float dy;
                
                if (dwidth <= vwidth && dheight <= vheight) {
                    scale = 1.0f;
                } else {
                    scale = Math.min((float) vwidth / (float) dwidth,
                            (float) vheight / (float) dheight);
                }
                
                dx = Math.round((vwidth - dwidth * scale) * 0.5f);
                dy = Math.round((vheight - dheight * scale) * 0.5f);

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(dx, dy);
            } else {
                // Generate the required transform.
                mTempSrc.set(0, 0, dwidth, dheight);
                mTempDst.set(0, 0, vwidth, vheight);
                
                mDrawMatrix = mMatrix;
                mDrawMatrix.setRectToRect(mTempSrc, mTempDst, scaleTypeToScaleToFit(mScaleType));
            }
        }
    }
```

分析一下上面的代码逻辑
1）当Drawable内部宽度或者高度为0，或者ScaleType.FIT_XY == mScaleType时
它的处理是
```java
mDrawable.setBounds(0, 0, vwidth, vheight);
mDrawMatrix = null;
```

为Drawable设置setBounds()，即设置Drawable的显示区域（如果我们在子类根据bounds的大小绘制），设置其显示区域为ImageView的大小，同时mDrawMatrix赋值为空，即不做变换处理。

2)当mScaleType为其他参数的时候，它会根据缩放模式，对mDrawMatrix进行赋值，赋值操作如缩小，偏移等。


所以我们再看到onDraw()方法体
ImageView#onDraw()
```java
   @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mDrawable == null) {
            return; // couldn't resolve the URI
        }

        if (mDrawableWidth == 0 || mDrawableHeight == 0) {
            return;     // nothing to draw (empty bounds)
        }

        if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
            mDrawable.draw(canvas);
        } else {
            int saveCount = canvas.getSaveCount();
            canvas.save();
            
            if (mCropToPadding) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;
                canvas.clipRect(scrollX + mPaddingLeft, scrollY + mPaddingTop,
                        scrollX + mRight - mLeft - mPaddingRight,
                        scrollY + mBottom - mTop - mPaddingBottom);
            }
            
            canvas.translate(mPaddingLeft, mPaddingTop);

            if (mDrawMatrix != null) {
                canvas.concat(mDrawMatrix);
            }
            mDrawable.draw(canvas);
            canvas.restoreToCount(saveCount);
        }
    }
```

如果mDrawMatrix为空，那么会直接绘制Drawable，直接绘制就是直接按照Drawable的内部宽度和内部高度来绘制，如果mDrawMatrix不为空，那么canvas会根据mDrawMatrix的数值进行变换，在变换的基础上绘制Drawable，这样绘制出来的视图则会根据ImageView的大小进行缩放。


从上面结果我们看出，ImageView在设置了Drawable之后，能够根据getIntrinsicWidth()以及getIntrinsicHeight()方法获取到Drawable的内部宽度和内部高度并且按照比例缩放显示。但是ImageView在scaleType为FIT_XY的情况下不会对mDrawMatrix进行赋值，如果我们在Drawable的子类中也没有根据bounds大小设置显示区域，在onDraw()方法体里，则会直接对Drawable进行绘制，也就不会有视图按比例缩放的效果，而可能只显示图片的一块区域。





## Drawable子类-BitmapDrawable
A Drawable that wraps a bitmap and can be tiled, stretched, or aligned. You can create a
BitmapDrawable from a file path, an input stream, through XML inflation, or from a {@link android.graphics.Bitmap} object.

看完了ImageView对于Drawable的处理，我们再看看继承Drawable的子类如何实现和处理视图绘制等方法，这里我们选择最常见的BitmapDrawable。

BitmapDrawable#draw()
```java
@Override
    public void draw(Canvas canvas) {
        final Bitmap bitmap = mBitmapState.mBitmap;
        if (bitmap == null) {
            return;
        }

        final BitmapState state = mBitmapState;
        final Paint paint = state.mPaint;
        if (state.mRebuildShader) {
            final Shader.TileMode tmx = state.mTileModeX;
            final Shader.TileMode tmy = state.mTileModeY;
            if (tmx == null && tmy == null) {
                paint.setShader(null);
            } else {
                paint.setShader(new BitmapShader(bitmap,
                        tmx == null ? Shader.TileMode.CLAMP : tmx,
                        tmy == null ? Shader.TileMode.CLAMP : tmy));
            }

            state.mRebuildShader = false;
        }

        final int restoreAlpha;
        if (state.mBaseAlpha != 1.0f) {
            final Paint p = getPaint();
            restoreAlpha = p.getAlpha();
            p.setAlpha((int) (restoreAlpha * state.mBaseAlpha + 0.5f));
        } else {
            restoreAlpha = -1;
        }

        final boolean clearColorFilter;
        if (mTintFilter != null && paint.getColorFilter() == null) {
            paint.setColorFilter(mTintFilter);
            clearColorFilter = true;
        } else {
            clearColorFilter = false;
        }

        updateDstRectAndInsetsIfDirty();
        final Shader shader = paint.getShader();
        final boolean needMirroring = needMirroring();
        if (shader == null) {
            if (needMirroring) {
                canvas.save();
                // Mirror the bitmap
                canvas.translate(mDstRect.right - mDstRect.left, 0);
                canvas.scale(-1.0f, 1.0f);
            }

            canvas.drawBitmap(bitmap, null, mDstRect, paint);

            if (needMirroring) {
                canvas.restore();
            }
        } else {
            if (needMirroring) {
                // Mirror the bitmap
                updateMirrorMatrix(mDstRect.right - mDstRect.left);
                shader.setLocalMatrix(mMirrorMatrix);
                paint.setShader(shader);
            } else {
                if (mMirrorMatrix != null) {
                    mMirrorMatrix = null;
                    shader.setLocalMatrix(Matrix.IDENTITY_MATRIX);
                    paint.setShader(shader);
                }
            }

            canvas.drawRect(mDstRect, paint);
        }

        if (clearColorFilter) {
            paint.setColorFilter(null);
        }

        if (restoreAlpha >= 0) {
            paint.setAlpha(restoreAlpha);
        }
    }
```

再看看getIntrinsicWidth()和getIntrinsicHeight()方法，它们返回Drawable的内部宽度和内部高度为Bitmap的宽度和高度。

BitmapDrawable#getIntrinsicWidth()

```java
    @Override
    public int getIntrinsicWidth() {
        return mBitmapWidth;
    }
```

BitmapDrawable#getIntrinsicHeight()

```java
    @Override
    public int getIntrinsicHeight() {
        return mBitmapHeight;
    }
```

# 关于自定义Drawable

在这里我使用Drawable来实现自定义视图效果，和自定义View一样，我们可以自定义Drawable来实现各种各种的动画效果，而且相对来说，步骤更加简洁方便。
1）重写getIntrinsicWidth()和getIntrinsicHeight()定义Drawable的内部高度
2）重写draw()方法，在方法体中绘制Drawable的内容

```java
/**
 * @author linzewu
 * @date 2016/11/27
 */
public abstract class AnimDrawable extends Drawable {
    @Override
    public void draw(Canvas canvas) {
        //绘制动画
    }

    @Override
    public void setAlpha(int alpha) {
             
    }

    @Override
    public void setColorFilter(ColorFilter colorFilter) {

    }

    @Override
    public int getOpacity() {
        return PixelFormat.TRANSLUCENT;
    }
    
    /* 当View设置为wrap_content时获取宽度值 */
    @Override
    public int getIntrinsicWidth() {
        return (int) mDrawableDesignWidth;
    }

    /* 当View设置为wrap_content时获取高度值 */
    @Override
    public int getIntrinsicHeight() {
        return (int) mDrawableDesignHeight;
    }
}
```

如上，这样我们就完成了一个自定义Drawable，不过它还是一个抽象类，具体实现我们会放到它的子类当中，然后为了更加方便的动画绘制。我们可以将一个Drawable视图分为若干个视图层，每一种动画效果我们都可以单独绘制成一个动画层。


```java
public abstract class AbsAnimLayer {
    
    /**
     * 绘制动画层大小
     * <br><strong>NOTE:</strong>   
     * @param designWidth Drawable内部宽度
     * @param designHeight Drawable内部高度
     */
    protected abstract void onMeasureLayer(int designWidth, int designHeight);

    /**
     * 绘制动画层
     * <br><strong>NOTE:</strong> 当动画线程每次刷新调用
     * @param canvas 画布
     * @param percent 动画百分比
     */
    protected abstract void onDrawLayer(Canvas canvas, float percent);
    
}
```

在draw()方法体里面，逐个遍历一个Drawable里面包含的所有动画层

```java
    @Override
    public void draw(Canvas canvas) {
        if (mDrawableDesignWidth == 0 || mDrawableDesignHeight == 0) {
            return ;
        }
        for (AbsAnimLayer absAnimLayer : mAbsAnimLayers) {
            absAnimLayer.onDrawLayer(canvas, mCurrentPercent);
        }
    }
```


# 动画库LoadingLib

上面实现了自定义Drawable，我们可以在上面绘制实现我们想要的各种动画效果，应用这种自定义动画视图的方式，来绘制动画。

这里我实现了一个加载动画库LoadingLib，里面集成了我目前模仿以及实现的动画加载效果，在后续的代码中，我会陆续模仿一些前段效果。

### 模仿SpinKit前端加载动画效果

![这里写图片描述](http://img.blog.csdn.net/20161230003328408?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20161230002849609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 如何使用

github地址：https://github.com/82367825/LoadingLib

引用库：
```
gradle版本

Step 1. Add the JitPack repository to your build file Add it in your root build.gradle at the end of repositories:

    allprojects {
        repositories {
            ...
            maven { url 'https://jitpack.io' }
        }
    }
Step 2. Add the dependency

    dependencies {
            compile 'com.github.82367825:LoadingLib:v1.1.0'
    }
```

使用例子：

```java
        SpinKitAnimDrawable spinKitAnimDrawable = new SpinKitAnimDrawable();
        spinKitAnimDrawable.setType(SpinKitAnimDrawable.TYPE_BOUNCE);
        mBounceImageView = (ImageView) findViewById(R.id.spinkit_bounce);
        mBounceImageView.setImageDrawable(spinKitAnimDrawable);
        spinKitAnimDrawable.startAnim();
```

未完待续...

