---
layout: post
title: Android-ShimmerLayout微光效果解析
categories: [Android]
description: 
tags: 微光效果
---

前阵子在github上看到一个很不错的动画效果，叫做SimmerLayout，是一个用于实现内部视图微光效果的布局。

![这里写图片描述](http://img.blog.csdn.net/20170814163637270?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 如何实现

通过使用PorterDuff，我们可以制造出微光效果。PorterDuff是canvas绘制图像处理中的一种渲染模式，当我们需要绘制出区域覆盖的图形效果的时候，我们可以使用这种方式来绘制。

这里我们采用的是PorterDuff.MODE.SRC_IN，意思是在绘制的时候，显示上下图层相交的部分，且这部分显示上层图层。

![这里写图片描述](http://img.blog.csdn.net/20170814163702934?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1） 首先我们需要绘制出最上层的微光，这里通过LinearGradient线性渐变渲染器来绘制微光渐变效果，为了使得渐变自然，我们看到，代码里在前后两端都加入了透明色。

```
private Bitmap getSourceMaskBitmap() {
        if (sourceMaskBitmap != null) {
            return sourceMaskBitmap;
        }

        int width = maskRect.width();
        int height = getHeight();
        
        /* 通过LinearGradient在遮罩Bitmap上绘制渐变效果 */
        final int edgeColor = reduceColorAlphaValueToZero(shimmerColor);
        LinearGradient gradient = new LinearGradient(
                -maskRect.left, 0,
                width + maskRect.left, 0,
                /* 透明色 - 微光颜色 - 微光颜色 - 透明色 */
                new int[]{edgeColor, shimmerColor, shimmerColor, edgeColor},
                new float[]{0.25F, 0.47F, 0.53F, 0.75F},
                Shader.TileMode.CLAMP);
        
        Paint paint = new Paint();
        paint.setShader(gradient);

        sourceMaskBitmap = createBitmap(width, height);
        /* 对微光效果的bitmap做一些旋转效果 */
        Canvas canvas = new Canvas(sourceMaskBitmap);
        canvas.rotate(shimmerAngle, width / 2, height / 2);
        canvas.drawRect(-maskRect.left, maskRect.top, width + maskRect.left, maskRect.bottom, paint);

        return sourceMaskBitmap;
    }
```

2） 然后，我们需要把微光效果的图层和视图本身的界面混合，使用PorterDuff.MODE.SRC_IN混合效果。首先如何绘制视图本身的界面，这里很简单，直接调用父类的绘制方法super.dispatchDraw(Canvas)，紧接着再绘制微光效果的图层。

```
/* 获取微光效果Bitmap */
localMaskBitmap = getSourceMaskBitmap();
canvas.save();
/* 先绘制GroupView本身的界面 */
super.dispatchDraw(canvas);
/* 再绘制微光效果Bitmap */
canvas.drawBitmap(localMaskBitmap, 0, 0, maskPaint);
canvas.restore();
```



3） 最后我们发现，微光效果是从左向右移动过去的，如何实现？

通过不断位移localMaskBitmap的位置，这里通过控制偏移值maskOffsetX，实现了微光慢慢位移的效果。

```
/* 获取微光效果Bitmap */
localMaskBitmap = getSourceMaskBitmap();
canvas.save();
/* canvas 裁剪显示 */
canvas.clipRect(maskOffsetX, 0,
                maskOffsetX + localMaskBitmap.getWidth(),
                getHeight());
/* 先绘制GroupView本身的界面 */
super.dispatchDraw(canvas);
/* 再绘制微光效果Bitmap */
canvas.drawBitmap(localMaskBitmap, maskOffsetX, 0, maskPaint);
canvas.restore();
```


这个动画原理本身很简单，不过在内存方面，因为创建了多个bitmap，如果当前界面不包含大图，对内存的消耗还是很低的，适用于为比较轻量级的界面添加效果。

更多细节，看看作者的原文介绍以及GitHub
>ShimmerLayout : https://medium.com/supercharges-mobile-product-guide/shimmerlayout-26978ab53c28
ShimmerLayout Github : https://github.com/team-supercharge/ShimmerLayout




