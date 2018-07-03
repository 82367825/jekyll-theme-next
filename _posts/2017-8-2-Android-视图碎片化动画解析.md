---
layout: post
title: Android-视图碎片化动画解析
categories: [Android]
description: 
tags: 动画解析
---

之前在网上看到一个动画效果，可以将布局内的视图实现碎片化，一直想知道怎么去实现，现在我们来解析一下这种效果究竟如何实现？

基础前提是我们需要先学习OpenGL, 在Android中有很多酷炫的动画效果，都是需要借助openGL来实现的，因此，学习这个图形库，也有助于增加我们开发技能。


>OpenGL csdn系列教程：http://blog.csdn.net/column/details/15997.html
OpenGL 投影理解：http://www.php361.com/index.php?c=index&a=view&id=6016

推荐通过上面博主的教程去学习OpenGL, 非常简单易懂的教程，因为一开始学习的时候，在网上一直找不到好的教程，所以走了很多弯路，在这里也感谢热心分享的博主。


>基本思路：既然是通过OpenGL来实现，那么肯定不是通过view层级去表现，我们的思路是根据提供的视图纹理图，通过OpenGL图形库来绘制碎片动画效果。


## 1 如何获取视图的纹理图

我们知道，View所有的绘制工作全都放在View#draw(Canvas canvas)方法里，那么当我们需要获取某一刻视图的界面图的时候，同理可以调用父布局的这个方法。

```
Bitmap bitmap = Bitmap.createBitmap(getWidth(), getHeight(), Bitmap.Config.ARGB_4444);
Canvas canvas = new Canvas(bitmap);
super.draw(canvas);
```


## 2 如何绘制碎片

### 1）单个碎片

在OpenGL中，图形绘制的基本单位是三角形，通过把任何目标图形分解成若干三角形，我们可以绘制任何我们想要实现的图形。在这里我们的每一个碎片都是一个矩阵，那么我们这样定义，每一个矩阵都是由两个三角形组成。

比如，假如我们现在只想绘制一个碎片，那么此时整个视图就是一个碎片。
那么它现在就是由V1V2V4以及V1V3V4两个三角形组成。

![这里写图片描述](http://img.blog.csdn.net/20170802153222720?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 2）碎片坐标

碎片坐标分为顶点坐标和纹理坐标，我们先确定顶点坐标，为了把视图分割为若干个碎片，我们需要按照上面定义单个碎片的方式，批量地生成碎片坐标，然后存放到数组中。

我们的顶点坐标系如下（忽略z坐标）

![这里写图片描述](http://img.blog.csdn.net/20170802152714435?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


那么我们可以得出每一个碎片的宽度值，以及高度值

```
碎片宽度值 = 2f / 横坐标碎片个数;
碎片高度值 = 2f / 纵坐标碎片个数;
```
依此，我们可以得出所有碎片的上四个坐标点（这里是6个坐标点，因为分割成两个三角形）的值。

```
/**
     * 初始化顶点坐标数据
     */
    private float[] initPositionData() {
        /* 每个碎片都是一个正方形, 由6个顶点决定, 每个顶点由x, y, z三个方向决定 */
        float[] positionData = new float[6 * 3 * mFragNumberX * mFragNumberY];

        float height = 1f;
        float width = height * mRatioValue;

        final float stepX = width * 2f / mFragNumberX;
        final float stepY = height * 2f / mFragNumberY;

        final float minPositionX = -width;
        final float minPositionY = -height;

        int positionDataOffset = 0;
        for (int x = 0; x < mFragNumberX; x++) {
            for (int y = 0; y < mFragNumberY; y++) {

                float z = (float) Math.random();
                
                final float x1 = minPositionX + x * stepX;
                final float x2 = x1 + stepX;

                final float y1 = minPositionY + y * stepY;
                final float y2 = y1 + stepY;

                // 定义一个碎片的四个顶点坐标
                final float[] p1 = {x1, y2, z};
                final float[] p2 = {x2, y2, z};
                final float[] p3 = {x1, y1, z};
                final float[] p4 = {x2, y1, z};

                int elementsPerPoint = p1.length;
                final int size = elementsPerPoint * 6;
                final float[] thisPositionData = new float[size];

                int offset = 0;
                // 构建一个碎片
                //  1---2
                //  | / |
                //  3---4
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p1[i];
                }
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p3[i];
                }
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p2[i];
                }
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p3[i];
                }
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p4[i];
                }
                for (int i = 0; i < elementsPerPoint; i++) {
                    thisPositionData[offset++] = p2[i];
                }

                System.arraycopy(
                        thisPositionData, 0,
                        positionData, positionDataOffset,
                        thisPositionData.length
                );
                positionDataOffset += thisPositionData.length;
            }
        }
        return positionData;
    }
```


### 3）碎片动画效果

前面我们已经实现了把视图分割成若干个碎片，那么现在就差动画效果。

在上面已经确定视图每一个碎片的坐标的前提下，我们需要做的是让碎片零散化，出现随意飘落的效果。

碎片的飘落效果分为x坐标的左移或者右移，y坐标的下移，z坐标的远近变化，这里我们需要一个动画进度值u_AnimationFraction，从外部传入，然后再构造一个随机值randomValue。

大致的预期效果如下：

![这里写图片描述](http://img.blog.csdn.net/20170802152644999?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


我们来看顶点着色器的代码：
```
uniform mat4 u_MVPMatrix;    //变换矩阵
uniform float u_AnimationFraction;  //动画进度
attribute vec4 a_Position;   //顶点坐标
attribute vec2 a_TexCoordinate; //纹理坐标

varying vec2 v_TexCoordinate;


void main() 
{
   //传递纹理坐标的值到纹理着色器
   v_TexCoordinate = a_TexCoordinate;
   
   vec4 finalPosition = a_Position;
   
   //制造一个随机漂移值
   float randomValue = (sin(u_AnimationFraction)) * (finalPosition.z - 0.5) * 0.2; 
   //碎片随机左移或者右移
   finalPosition.x = finalPosition.x + randomValue * 0.2;
   //碎片随机加速
   finalPosition.y = finalPosition.y - u_AnimationFraction - finalPosition.z * 0.2 * u_AnimationFraction;
   finalPosition.z = finalPosition.z * (u_AnimationFraction / 2.0) * 0.1;
   
   gl_Position = u_MVPMatrix * finalPosition;
}
```

## 3 最终效果

首先我们需要加入相关的权限

```
    <uses-feature android:glEsVersion="0x00020000" android:required="true" />
```

我们在布局中，加入我们想要实现碎片化动画的控件。

```
<com.zero.fragmentanimation.openGL.ShatterAnimLayout
        android:id="@+id/main_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <LinearLayout
            android:background="@drawable/main_bg"
            android:orientation="vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent">    
            <Button
                android:id="@+id/main_button"
                android:text="start animation"
                android:layout_gravity="center_horizontal"
                android:layout_margin="60dp"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
        </LinearLayout>
 </com.zero.fragmentanimation.openGL.ShatterAnimLayout>

```

点击开始动画，看到最后的动画效果

![这里写图片描述](http://img.blog.csdn.net/20170802152532178?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当我们控制碎片数量足够多的时候，比如我们设置横坐标方向上的碎片个数为100

![这里写图片描述](http://img.blog.csdn.net/20170802152606571?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


我们看到，这时候，每个碎片的显示效果接近一个小点，和我们经常看到的粒子效果非常相似。（粒子效果，是通过获取View截图上每一个像素点的颜色值，然后根据这些像素颜色值，绘制出若干粒子爆炸的效果，相关的效果介绍：<a href="http://blog.csdn.net/crazy__chen/article/details/50149619/">粒子效果介绍</a>）

<br>

>动画解析完了，最后是解析动画的demo代码：https://github.com/82367825/OpenGLShatterAnim

<br>
>这篇博客的解析效果参考了Yalantis的一个动画效果，如果感兴趣的可以去看。学习Yalantis的开源库非常有意思也有收获。
>
>Yalantis上的一个碎片化动画效果库 <a href="https://github.com/Yalantis/StarWars.Android">StarWars.Android</a>


