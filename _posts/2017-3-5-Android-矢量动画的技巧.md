---
layout: post
title: Android-矢量动画的技巧
categories: [Android]
description: Android5.0之后加入的矢量动画
tags: SVG 
---
# 前言

>矢量动画，自从Android5.0之后加入对矢量动画的支持，越来越多的矢量动画开始应用到各个场景中，不同于传统的Canvas动画，虽然在应用面上更多体现在icon
轻量级动画上，但是我们依旧有学习它的必要。在看了国外的开发者Alex Lockwood的矢量动画技巧之后，感觉学到很多，本文基本上是对Alex Lockwood发布博文的翻译以及补充一些自己的理解。

如果希望看Alex Lockwood的英文博客原始的内容链接可以点击这里：
[链接](http://www.androiddesignpatterns.com/2016/11/introduction-to-icon-animation-techniques.html)

# 关于Vector
Android5.0之后加入了对Vector的支持，通过VectorDrawable来实现矢量图。矢量图的好处我们都知道，在可以大量减少图像的体积的同时，还能够自使用缩放，保证图像不失真。而在Android之下，我们还可以通过AnimatedVectorDrawable来实现矢量图和属性动画之间的联系，通过几行简单的代码，实现复杂的矢量动画。

### vector标签
vector标签用于实现一个矢量图，我们可以通过在这个xmL文件中编写矢量图绘制代码，来实现矢量图，一般情况下，除了简单的图像我们自己去实现绘制代码，一些复杂的图像，我们可以直接通过设计工具导出的矢量图去获取数据

### animated-vector标签
animated-vector标签作为矢量图和属性动画之间的连接，当我们想要为矢量图中任意一个部分设置属性动画的时候，我们需要在animated-vector标签中设置target子标签，为其设置目标属性动画。

# 关于Vector动画技巧

如何很好的利用动态Vector来实现美观的动画效果，我们可以分为几个步骤
>* 绘制Path
>* 组合Path
>* Path修整
>* Path形变
>* Path裁减

### 绘制Path
在开始矢量动画学习之前，我们首先需要了解矢量图是如何绘制的。在Android中，我们通过VectorDrawable来实现矢量图的绘制，VectorDrawable在概念上和web端的SVG很相似，它们都是通过绘制一系列的Path来实现图像绘制。


| 操作命令        | 描述    |
| --------   | -----  |
| M x,y       | 位移到点 (x,y) |  
| L x,y       | 绘制一段直线到（a,b） |  
| C a1,b1,a2,b2,x1,y1| 以（a1,b1）(a2,b2) 为控制点绘制三阶贝塞尔曲线|
| Q a1,b1,x1,y1| 以（a1,b1）为控制点绘制二阶贝塞尔曲线|
| Z |  关闭路径|      

<br>
所有的Path都有两种表现形式：filled 和 stroked. 如果path的表现形式是filled，那么它的绘制的形状将会被填充绘制。如果path的表现形式是stroked，那么将会沿着绘制的形状描边。

同时这两种表现形式的path都有它们自己的属性参数

| Property name      | Element type | Value type |Min|Max|
| --------   | -----  | --- | --- | --- |
| android:fillAlpha | path | float | 0 | 1 |
| android:fillColor | path | integer | --- | --- |
| android:strokeAlpha | path | float | 0 | 1 |
| android:strokeColor | path | integer | --- | --- |
| android:strokeWidth | path | float | 0 | --- |

<br>

让我们来看一个例子，我们希望绘制一个播放和暂停的icon，我们可以通过path为每一个状态绘制视图。

```xml
<vector
  xmlns:android="http://schemas.android.com/apk/res/android"
  android:width="48dp"
  android:height="48dp"
  android:viewportHeight="12"
  android:viewportWidth="12">

  <!-- This path draws an orange triangular play icon. -->
  <path
    android:fillColor="#FF9800"
    android:pathData="M 4,2.5 L 4,9.5 L 9.5,6 Z"/>

  <!-- This path draws two green stroked vertical pause bars. -->
  <path
    android:pathData="M 4,2.5 L 4,9.5 M 8,2.5 L 8,9.5"
    android:strokeColor="#0F9D58"
    android:strokeWidth="2"/>

  <!-- This path draws a red circle. -->
  <path
    android:fillColor="#DB4437"
    android:pathData="M 2,6 C 2,3.8 3.8,2 6,2 C 8.2,2 10,3.8 10,6 C 10,8.2 8.2,10 6,10 C 3.8,10 2,8.2 2,6"/>

</vector>
```

<img src="http://img.blog.csdn.net/20170305232134379?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="250" height="400" 
 alt="image" align="left"/>

<img src="http://img.blog.csdn.net/20170305232321990?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="250" height="400" 
 alt="image"/>


<br/>
<img src="http://img.blog.csdn.net/20170305232754757?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="250" height="400" 
 alt="image"/>

我们看到android:width和android:height代表矢量图的实际大小，android:viewportWidth和
android:viewportHeight则代表我们在矢量图绘制上，划分的尺寸数量，例如这里的12*12，则代表矢量图被划分为12*12的网格，在绘制的时候，默认的长度单位就是一个网格的大小。

像一开始我们提到的，VectorDrawable的一个优势就是它不依赖显示的屏幕密度尺寸，而会根据大小而缩放显示。有了这个好处，我们就可以不再为了不同手机屏幕尺寸而做不同的png图来适配，不仅可以省去我们许多适配机型的事件，又可以缩减apk包体。


### 组合Path

通过前面的例子，我们可以看出，path绘制的矢量图外观，是直接受到颜色，大小，路径等因素的影响的。除了这些影响因素，VectorDrawable还为我们提供了一系列的转变操作，例如，位移，旋转，缩放等操作，我们可以通过group标签来组合若干path，同时执行这些操作。

|Property name|Element type|Value type|
| ---- |---- | ---- |
|android:pivotX|group|float|
|android:pivotY|group|float|
|android:rotation|group|float|
|android:scaleX|group|float|
|android:scaleY|group|float|
|android:translateX|group|float|
|android:translateY|group|float|

<br/>
对于group标签，我们需要知道两个重要的点：

（1）子group会继承父group的变换操作

（2）同一层级的group标签内，变换操作的执行顺序是，scale，rotation，translation

通过group标签，我们可以对一组的Path矢量图对象执行变换操作，我们在上面的播放矢量图做变换操作：

```xml
<!-- Translate the canvas, then rotate, then scale, then draw the play icon. -->
  <group android:scaleX="1.5" android:pivotX="6" android:pivotY="6">
    <group android:rotation="90" android:pivotX="6" android:pivotY="6">
      <group android:translateX="2">
        <path android:name="play_path"/>
      </group>
    </group>
  </group>
```

<img src="http://img.blog.csdn.net/20170305233214557?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="250" height="400" alt="图片名称"/>

通过group标签组合path实现转换效果的能力，我们可以实现一些简洁又好看的动画。
例如下面这个闹钟动画，我们只需要对闹钟头部区域设置一段旋转的变换动画，就能实现这种动画效果。

![这里写图片描述](http://img.blog.csdn.net/20170305233938107?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

又如下面这个水平进度条动画效果，通过位移变换和缩放变换的组合使用，实现一个效果流畅的水平进度加载的效果。

![这里写图片描述](http://img.blog.csdn.net/20170305234122795?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)




### Path修整

Stroked Path最不被知晓的一个属性就是它能够被修整，也就是说，我们可以使得一个Stroked Path有选择性地显示一部分。在Vector中，我们可以通过以下属性来实现这一个功能：

|Property name|Element type|Value type|Min|Max|
|----|----|----|----|----|
|android:trimPathStart|path|float|0|1|
|android:trimPathEnd|path|float|0|1|
|android:trimPathOffset|path|float|0|1|

<br/>
trimPathStart参数决定了Path从哪里开始显示，trimPathEnd参数决定了Path到哪里结束显示，而trimPathOffest参数决定了Path显示区域的偏移距离。

我们看下面两个例子：


```xml
<vector
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="48dp"
    android:height="48dp"
    android:viewportHeight="12"
    android:viewportWidth="12">

    <path
        android:strokeColor="#CCCCCC"
        android:strokeWidth="0.4"
        android:pathData="M 0,6 L 12,6"
        />

    <path
        android:strokeColor="#121323"
        android:strokeWidth="0.4"
        android:pathData="M 0,6 L 12,6"
        android:trimPathStart="0"
        android:trimPathEnd="0.5"
        android:trimPathOffset="0"
        />

</vector>
```
android:trimPathStart="0"
android:trimPathEnd="0.5"
android:trimPathOffset="0"

效果图：

<img src="http://img.blog.csdn.net/20170305234354921?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="400" height="100" alt="图片名称"/>


```xml
<vector
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="48dp"
    android:height="48dp"
    android:viewportHeight="12"
    android:viewportWidth="12">

    <path
        android:strokeColor="#CCCCCC"
        android:strokeWidth="0.4"
        android:pathData="M 0,6 L 12,6"
        />

    <path
        android:strokeColor="#121323"
        android:strokeWidth="0.4"
        android:pathData="M 0,6 L 12,6"
        android:trimPathStart="0"
        android:trimPathEnd="0.5"
        android:trimPathOffset="0.2"
        />

</vector>
```
android:trimPathStart="0"
android:trimPathEnd="0.5"
android:trimPathOffset="0.2"

效果图：

<img src="http://img.blog.csdn.net/20170305234519286?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="400" height="100" alt="图片名称"/>

这样的修整，可以使得Path的使用更加灵活，通过对这个属性对一些属性动画，我们可以实现更加酷的效果

1）指纹动画，由5个Stroked Path组合而成，每个Path的trimPathStart和trimPathEnd的值分别被设置为0和1。当指纹隐藏的时候，值会动态地变化为0；当指纹显示的时候，值会动态地变化为1。我们看到另一个动画效果，草书动画，也是同样的原理，唯一不同的是，草书动画是让组合而成的Path，按照顺序执行，而不是同时进行。

效果图：

![这里写图片描述](http://img.blog.csdn.net/20170305234807189?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里同理，我们来实现一个草书动画：

![这里写图片描述](http://img.blog.csdn.net/20170306130824611?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

然后依样画葫芦，我给自己写了个草书签名（写的很丑）：

![这里写图片描述](http://img.blog.csdn.net/20170306130854549?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2）搜索返回动画，由几段Stroked Path巧妙的组合而成，无缝地完成搜索图标转变为返回图标的过渡动画。实现这种效果实际上十分简单，我们只需要在另一个Stroked Path 动画开始的时候，为trim的属性动画设置一个延迟启动时间，这样在视觉上我们就会以为这是一个连续的动画效果。

效果图：

![这里写图片描述](http://img.blog.csdn.net/20170305235442333?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3）Google IO 2016 icon，每一个动画数字由4段Stroked Path组合而成，每一段Stroked Path都有不同的颜色和动画数字总长的1/4长度，这里的动画效果是通过对每一段Stroked Path的trimPathOffset属性做从0到1的属性动画而实现的。

效果图：

![这里写图片描述](http://img.blog.csdn.net/20170305235755506?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Path形变

现在我们来看矢量动画中，最为精彩和有趣的技巧，就是Path Morphing(路径形变)。这种动画效果目前仅仅支持Android5.0以及更高版本，Path Morphing允许我们实现两段Path之间的无缝转换，这种动画效果的转换通过标签pathData来实现。

|Property name|Element type|Value type|
| --- | --- |---|
|android:pathData|path|string|

<br/>

为了实现Path A到Path B的形变动画，我们需要满足以下条件：

1）A 和 B 拥有相同数量的Path绘制命令

2）A 中的第i条绘制命令的种类必须和 B中的第i条绘制命令相同

3）A 中的第i条绘制命令的参数数量必须和 B中的第i绘制命令的参数数量相同

当我们没有满足以上任何一个条件的时候，在运行动画的时候，会出现运行崩溃的情况。在动画绘制开始前，framework从每个Path的andorid:pathData参数提取绘制命令的种类和他们的坐标位置，如果这时候上面的条件满足了，framework会认定两个Path之间的不同仅仅只是他们绘制命令中嵌入的坐标位置，此时framework会继续执行原来的Path的绘制命令，同时根据此时的动画进度，来计算出需要被替换的坐标值。

例如，Path A的pathData为 M 0,0..., Path B的pathData为 M 5,5 ...，这时候根据上面的描述，从Path A转化为Path B的时候，framework会继续执行原来的绘制命令M x,y，这时候坐标就要根据动画的进度来改变，当动画进度为0.2的时候，执行的命令就是M 1，1.

我们来看下面这个数字形变动画（个人很喜欢哈哈）

![这里写图片描述](http://img.blog.csdn.net/20170306000258529?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


虽然从概念上讲起来简单，但是实际上，形变动画的实现是非常耗时的，在这期间最耗时的动作就是为了让这个形变动画看起来更加自然，我们需要手动反复地调节Path形变的动画的开始状态和结束状态。我们可以通过这些技巧来使得动画看起来自然顺畅。

1）添加多余的绘制坐标
例如下面这个加号形变成减号的动画，我们绘制一个矩阵型减号只需要4条绘制命令，然而我们绘制一个加号却需要12条绘制命令，为了使得这两个图形能够兼容形变动画，我们需要在绘制减号的时候，添加多余8个空绘制命令。

![这里写图片描述](http://img.blog.csdn.net/20170306000554088?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

加号绘制代码：
```java
M 5,11 L 11,11 L 11,5 L 13,5 L 13,11 L 19,11 L 19,13 L 13,13 L 13,19 L 11,19 L 11,13 L 5,13 Z
```
减号绘制代码：
```java
M 5,11 L 11,11 L 11,11 L 13,11 L 13,11 L 19,11 L 19,13 L 13,13 L 13,13 L 11,13 L 11,13 L 5,13 Z
```
比较之后我们不难看出，减号的绘制中出现了一些无意义的绘制命令，这就是我们需要的。

2）巧妙运用贝塞尔曲线绘制命令
三阶的贝塞尔绘制命令除了可以绘制各种曲线，当然也是可以用来绘制直线的，只要我们保持它的两个控制点分别和起点以及终点相等即可。当我们需要从L绘制命令形变成C绘制命令的时候，理论上这是不符合绘制条件的，但是当我们用C命令来绘制直线的时候，问题就解决了。
三阶贝塞尔曲线除了可以绘制直线，还可以绘制圆弧。
Alex Lockwood已经在github为我们提供了代码https://gist.github.com/alexjlockwood/8ca9228e861222866c666513ed401a71

有时候，我们直接需要的是绘制圆形，这个在之前的博客中我有提到


3）有时候无论我们做了什么调整，形变动画依旧看起来别扭。我们可以添加一个180度或者360度的旋转动画，为什么这样做呢，因为旋转动画可以分散眼睛的注意力，导致观看者注意力不会一直在形变动画上，同时也使得这个动画效果显得更加敏捷。

4）我们需要知道，形变动画最终还是要由绘制命令中的坐标间的相对距离决定的。我们需要在可能的情况下，尽可能地缩小绘制命令之间的坐标的距离，保证动画看起来更加自然，顺畅。


### Path裁剪

最后的技巧我们来看标签clip-path的使用，clip-path会限制一个区域，只有绘制在这块区域的Path绘制，才会显示，而其他在这块区域之外的绘制，都不会显示。这有点类似我们canvas2D绘制中的Canvas#clipPath()方法。

|Property name|Element type|Value type|
| --- | --- |---|
|android:pathData|clip-path|string|

<br/>
通过限定显示的区域，我们可以实现很多酷炫的效果。尤其在有填充动画效果的动画中，clipPath是非常有用的。

![这里写图片描述](http://img.blog.csdn.net/20170306000818427?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


通过clipPath限制了绘制的显示区域，我们看到下图，其实clipPath绘制出的限制区域如下，黑色区域则为可显示区域，因此眼睛图形的部分区域则会被隐藏。

<img src="http://img.blog.csdn.net/20170306125831435?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="300" height="200" alt="图片名称"/>


还有类似这样的：

![这里写图片描述](http://img.blog.csdn.net/20170306001236136?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

<br/>

![这里写图片描述](http://img.blog.csdn.net/20170306001355966?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

<br/>
<br/>

### Path整合应用
最后，我们通过一个综合的例子，来使用上面提到六种矢量动画的技巧

1）Fill alpha
2）Stroke width
3）Translation and rotation
4）Trim path start/end
5）Path morphing
6）Clip path

效果图：
<br/>

![这里写图片描述](http://img.blog.csdn.net/20170306001701594?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#关于Vector的兼容
Vector动画这么好用，但是当它刚被推出的时候，由于兼容性的问题，基本很少被使用的。一开始，安卓中的矢量图，只支持Android5.0以及更高版本，但是后来Google推出了com.android.support:appcompat-v7:23.2.0兼容库，我们只需要引入这个版本以及以上的兼容库，就可以在低版本兼容Vector的使用，最低能够兼容到Android2.1，现在基本也没有应用会低于这个api版本了。

那么如何实现兼容呢？

1）
使用Gradle Plugin 2.0以上的情况
Gradle2.0之后我们直接支持通过兼容库来解决
```gradle
android {

    defaultConfig {
        vectorDrawables.useSupportLibrary = true
    }
}
```
使用Gradle Plugin 2.0以下，Gradle Plugin 1.5以上的情况，我们需要加入以下声明
因为Gradle1.5开始加入对矢量图的兼容,但并不是通过兼容库,而是靠gradle把矢量图生成png图, 
generatedDensities实际上就是要生成PNG的图片分辨率的数组，使用appcompat后就不需要这样了.
```gradle
android {
  defaultConfig {
    // Stops the Gradle plugin’s automatic rasterization of vectors
    generatedDensities = []
  }
  // Flag to tell aapt to keep the attribute ids around
  aaptOptions {
    additionalParameters "--no-version-vectors"
  }
}
```

2）如果我们是通过兼容库来解决兼容问题, 加入兼容库的引用
```
compile 'com.android.support:appcompat-v7:23.4.0'
```

Activity继承自AppcompatActivity

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}

```

通过这些兼容措施，我们就能够在2.1以上的版本中使用矢量图了，但是矢量动画的兼容依然存在问题，我们会发现，在低于Android5.0的机型上，path morphing（路径形变动画）以及path interpolar（路径插值器）是无法使用的。
其他更多的兼容问题，我们可以看大神博客：
（ Android Vector曲折的兼容之路）http://blog.csdn.net/eclipsexys/article/details/51838119



# 关于Vector以及SVG


兼容了Vector的使用，以及学到了Vector动画的使用技巧，但是我们想要画出想要的矢量图，依旧是需要一盘波折的。我们绘制矢量图，矢量图绘制路径又复杂，这时候怎么办，肯定不是自己去计算，那样太费时间又不现实。

1）求助设计师，或者自己兼职设计师，通过ps等设计软件设计好图形，并导出成svg图。

2）对于简单的图形，我们可以自己设计
在线的SVG编辑器：
https://svg-edit.github.io/svgedit/releases/svg-edit-2.8.1/svg-editor.html

对于复杂的图形，如果我们手头有图，但是不是SVG图，那么我们可以通过这个在线网站转换
在线的SVG转化器:
ttps://convertio.co/zh/jpg-svg

还有更多关于SVG技巧的链接：
http://www.oschina.net/translate/20-useful-svg-tools-for-better-graphics

最后，在开发的时候，我们更可以直接通过AndroidStudio来获取矢量图：
右键工程-新建-Vector Asset
我们可以点击选择系统以及为我们提供了矢量图
<br/>

![这里写图片描述](http://img.blog.csdn.net/20170306002032256?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



我们可以看到，刚刚上面的闹钟动画，也是来自于这里的默认矢量图，基本满足常用的开发，矢量图的应用范围十分广泛，小到上面的一系列icon，大到复杂的图形，只要我们有了相应的绘制数据。

就像下面的复杂图形，我们也可以绘制出来，同时加上相应的动画效果。

![这里写图片描述](http://img.blog.csdn.net/20170306002413628?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvejgyMzY3ODI1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


# 最后

矢量图动画在Android端的应用目前还没有web端的广泛，但是随着它的优势越来越明显，相信会被越来越多主流应用采用。矢量动画主要应用于icon等轻量级的视图，当面对交互复杂的动画需求时，我们还是需要通过比较传统的ViewGroup内部处理动画逻辑。

感谢Alex Lockwood的博文，让我对矢量动画进一步的理解，这里可能翻译有些偏差，也是希望做一次完整的记录。

demo地址：https://github.com/82367825/AmazingAnim

------
参考：
http://blog.csdn.net/eclipsexys/article/details/51838119
http://www.androiddesignpatterns.com/2016/11/introduction-to-icon-animation-techniques.html
https://github.com/alexjlockwood/adp-delightful-details

