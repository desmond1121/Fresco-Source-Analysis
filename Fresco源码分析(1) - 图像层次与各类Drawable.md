#Fresco源码分析(1) - 图像层次与各类Drawable
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)


首先介绍几种Fresco中的图像层次，了解它们会帮助你理解Fresco加载图像的原理。

##1 引论：给图像分层次是什么作用？

如果你使用过Fresco这个强大的库之后，你就知道它可以在一个图像的加载、绘制过程中实现极大的定制化。你可以设置进度条来显示图片加载/下载的进度，可以设置占位图等到图片加载/下载成功后再显示目标图片，可以让在加载/下载失败后显示失败图片（更多功能参考[Fresco中文文档](http://fresco-cn.org/docs/)）。Fresco将进度条、占位图、失败图都作为图像的一层视图来管理，这部分仅仅负责视图层次绘制，将负责视图功能部分与逻辑部分尽可能实现解耦。

Fresco中定义了许多Drawable，它们都直接或间接继承了`Drawable`，但是各自的功能是不一样的。经过总结，我认为其中一共有三种功能的Drawable：层次型、容器型和视图型。直接上源码的一个视图例子就好理解了，作者进行了适当修改与翻译。

 o 层次型Drawable（维持图层）<br/> |<br/>------ 容器型Drawable（可对内容进行缩放）<br/> |　　　　|<br/>|　　　　--- 视图型Drawable（存放占位图）<br/> |<br/> ------ 容器型Drawable（可对内容进行缩放）<br/>|　　　　|<br/>|　　　　----- 容器型Drawable（可多次设置内容）<br/> |　　　　　　　|<br/> |　　　　　　　--- 视图型Drawable（存放目标显示图片）<br/> |<br/>------ 容器型Drawable（可对内容进行缩放）<br/>|　　　　|<br/>|　　　　--- 视图型Drawable（存放重试图片）<br/>|<br/>------ 容器型Drawable（可对内容进行缩放）<br/> 　　　　|<br/>  　　　　--- 视图型Drawable（存放失败图片）

该例位于`com.facebook.drawee.generic.GenericDraweeHierarchy`的类注释中。

这个例子充分描述了一个图像的层次，当然也可以在设置的时候往里面自行设置所需要的图层。

##2 层次型Drawable

在这一节中介绍的Drawable并不直接负责具体图像绘制，而是负责组建图像层次。

###2.1 ArrayDrawable

`ArrayDrawable`内部存储着一个Drawable数组，它与Android内置的`LayerDrawable`很相似，可见它将数组中的`Drawable`当做它的图层，在绘制的时候`ArrayDrawable`会按照数组顺序绘制其中的图层，数组最后的成员会显示在最上方。不过与`LayerDrawable`最大的不同的点有两处：
- 绘制顺序虽然是数组顺序，但是`ArrayDrawable`在绘制时会跳过暂时不需要绘制的图层；
- 在`ArrayDrawable`中不支持动态的添加/删除图层，只能在初始化时通过传入的数组决定图层数。不过好在它能够为存在的图层更换Drawable。（关于`LayerDrawable`可以参考我翻译的一文章：[Android LayerDrawable](http://blog.csdn.net/desmondj/article/details/47751553)。）

###2.2 FadeDrawable

`FadeDrawable`继承了`ArrayDrawable`。它除了具有`ArrayDrawable`本身的功能之外，还提供隐藏/显示图层的功能（可设置渐变）。具体的几个核心函数有：

 * `setTransitionDuration(int durationMs)` 设置隐藏/显示图层渐变动画时间（默认为300ms）。
 * `fadeInLayer(int index)` 显示指定图层
 * `fadeOutLayer(int index)` 隐藏指定图层
 * `fadeInAllLayers()`  显示所有图层
 * `fadeOutAllLayers()`  隐藏所有图层
 * `fadeToLayer(int index)` 显示指定图层同时隐藏其他图层
 * `fadeUpToLayer(int index)` 隐藏数组下标<=index的图层
 
它内部维护着一个boolean数组来维持需要显示的图层（可以调用`isLayerOn(int inxex)`查看指定图层是否显示）。
 
 
##3 容器型Drawable
 
**ForwardingDrawable**

`ForwardingDrawable`通俗的来说就是图片容器。它内部维护一个`Drawable`变量`mCurrentDelegate`，将Drawable的基本函数以及一些回调函数传递给目标图片，并在`draw(Canvas)`函数中调用`mCurrentDeletate.draw(Canvas)`函数将目标图片绘制出来。
它可以通过`getCurrent()`来获取容器内容，起到一个相当于是传递树的作用。它是所有容器型Drawable的基类，以下介绍几个它的子类，他们实现了不同功能的容器包装。

**ScaleTypeDrawable**

`ScaleTypeDrawable`封装了对代理图片的缩放处理，具体的缩放参数（`ScaleType`）与[Android ScaleType](http://developer.android.com/reference/android/widget/ImageView.ScaleType.html)的名字、功能相同。它在处理图片缩放的时候与`ImageView`的处理方式相似，我们来看一下它是怎么处理的：

```java
  private void configureBoundsIfUnderlyingChanged() {
    /* 当图像尺寸改变后，就重新确定图像边缘 */
    if (mUnderlyingWidth != getCurrent().getIntrinsicWidth() ||
        mUnderlyingHeight != getCurrent().getIntrinsicHeight()) {
      configureBounds();
    }
  }
```
其中`mUnderLyingWidth`与`mUnderLyingHeight`维护了已知的上一张图片宽高（初始均为0），当要绘制时调用这个函数，如果与上一张绘制的图像不一样时，就重新确认绘制边缘与缩放矩阵。

再来看一下`configureBounds()`是怎么确定转换矩阵的吧：
```java
 void configureBounds() {
    ...
    // 特殊情况判断：绘制图片是否为空白图或与之前绘制的图片尺寸相同
    ...
    
    // 当要往X、Y方向上填充容器时，直接将目标图片边界设置成容器图片的边界即可，Drawable会在绘制的时候自己调整。
    if (mScaleType == ScalingUtils.ScaleType.FIT_XY) {
      underlyingDrawable.setBounds(bounds);
      mDrawMatrix = null;
      return;
    }
    //处理其他缩放情况
    underlyingDrawable.setBounds(0, 0, underlyingWidth, underlyingHeight);
    ScalingUtils.getTransform(
        mTempMatrix,
        bounds,
        underlyingWidth,
        underlyingHeight,
        (mFocusPoint != null) ? mFocusPoint.x : 0.5f,
        (mFocusPoint != null) ? mFocusPoint.y : 0.5f,
        mScaleType);
    mDrawMatrix = mTempMatrix;
  }
```
  这里通过`ScalingUtils.getTransform`来计算出变换矩阵，我们以`CENTER_INSIDE`为例研究一下它的工作机制：

```java
public static Matrix getTransform(
      final Matrix transform,
      final Rect parentBounds,
      final int childWidth,
      final int childHeight,
      final float focusX,
      final float focusY,
      final ScaleType scaleType) {
    ...
    
    final float scaleX = (float) parentWidth / (float) childWidth;
    final float scaleY = (float) parentHeight / (float) childHeight;
    ...
    
    switch(scaleType){
		...
		case CENTER_INSIDE:
		        //计算缩放倍数
			    scale = Math.min(Math.min(scaleX, scaleY), 1.0f);
			    //计算平移距离
		        dx = parentBounds.left + (parentWidth - childWidth * scale) * 0.5f;
		        dy = parentBounds.top + (parentHeight - childHeight * scale) * 0.5f;
		        //设置缩放矩阵
		        transform.setScale(scale, scale);
		        //设置评议距离
		        transform.postTranslate((int) (dx + 0.5f), (int) (dy + 0.5f));
			break;
	}
	return transform;
}
```
其他具体的`ScaleType`处理就不赘述了，有兴趣的同学可以自己看源码再研究。**默认的ScaleType是CENTER_CROP。**

在计算好矩阵之后，我们来看一下这个容器是怎么将它的内容绘制出来的：

```java
  public void draw(Canvas canvas) {
    //计算矩阵
    configureBoundsIfUnderlyingChanged();
    if (mDrawMatrix != null) {
      int saveCount = canvas.save();
      canvas.clipRect(getBounds());
      canvas.concat(mDrawMatrix);
      super.draw(canvas);
      canvas.restoreToCount(saveCount);
    } else {
      //无变换矩阵时，直接让Drawable绘制到确定的边缘中。
      super.draw(canvas);
    }
  }
```
我们可以看到它将矩阵应用到`Canvas`中，并调用`ForwardingDrawable`的`draw(Canvas)`让它将目标视图绘制出来，之后还原Canvas的缩放属性防止累加缩放。

*暂时先介绍各类容器Drawable的功能，为方便后续理解。待分析完Fresco的架构之后再为分析。*

- **SettableDrawable**：可以多次设置内容Drawable的容器，多用在目标图片的图层中。
- **AutoRotateDrawable**：提供内容动态旋转的容器。
- **OrientedDrawable**：可以将内容Drawable以一个特定的角度绘制的容器。
- **MatrixDrawable**：可以为内容应用变形矩阵的容器，它只能赋予给显示目标图片的那个图层。***不能在一个图层上同时使用MatrixDrawable与ScaleTypeDrawable！***
- **RoundedCornersDrawable**：可以将内容的边界修剪成圆角矩形（目前版本暂不支持）或用实心的圆角矩形覆盖内容的容器。
- **GenericDraweeHierarchy.RootDrawable**：专门用于顶层图层的容器。

##4 视图型Drawable

大多数情况下，Fresco用于表现图片的视图型Drawable使用的就是Android原生`Drawable`来做图像的载体。不过也有两个例外：

**ProgressBarDrawable**

`ProgressBarDrawable`是负责绘制进度条的Drawable。它内部维持一个`level`用来描述进度(0<=level<=10000)，并自己实现了绘制过程，我们首先通过源码来看一下是怎么绘制的：

```java
  @Override
  public void draw(Canvas canvas) {
    if (mHideWhenZero && mLevel == 0) {
      return;
    }
    drawBar(canvas, 10000, mBackgroundColor);
    drawBar(canvas, mLevel, mColor);
  }

  private void drawBar(Canvas canvas, int level, int color) {
    Rect bounds = getBounds();
    int length = (bounds.width() - 2 * mPadding) * level / 10000;
    int xpos = bounds.left + mPadding;
    int ypos = bounds.bottom - mPadding - mBarWidth;
    mPaint.setColor(color);
    canvas.drawRect(xpos, ypos, xpos + length, ypos + mBarWidth, mPaint);
  }
```
可以看出，它先将整个进度条填充满`backgroundColor`颜色(可以通过`setBackgroundColor`设置)，再将进度覆盖区域矩形填充满`color`颜色(可以通过`setColor`设置)。

**RoundedBitmapDrawable**

这个Drawable与上面的容器型`RoundedCornersDrawable`有几个区别：

`RoundedBitmapDrawable`是将自身内容修剪成圆角矩形边绘制出来，并且可以使用Bitmap作为对象，返回一个`BitmapDrawable`。而`RoundedCornersDrawable`是将容器内容修剪成圆角矩形边，并且可以选择是否用指定颜色覆盖容器内容，可以使用任何Drawable当做容器。

*待分析完Fresco的架构之后，会回来分析圆角图片的实现机制。它与正常的使用Xfermode实现方式不同。*

实际上这两个类的功能是有一定重合的，我认为是由于`RoundedCornerDrawable`目前只能做到用圆角矩形覆盖内容，而无法将内容修剪成圆角矩形，所以才使用了`RoundedBitmapDrawable`。关于`RoundedCornersDrawable`的功能Fresco也在改进中。期待后续它能将两个功能合并起来。

##5 特殊Drawable - TransformAwareDrawable 和 VisibilityAwareDrawable

为什么说它们特殊呢，因为他们只是接口！

`TransformAwareDrawable `要和`TransfromCallback`一起使用。`TransformAwareDrawable `的作用很简单，就是提供设置`TransfromCallback`的回调函数，那我们来看看`TransfromCallback`的作用是什么：

```
public interface TransformCallback {

  // 获取已经应用在自身的变换Matrix，储存在transfrom中。
  public void getTransform(Matrix transform);

  // 获取根节点边界，储存在bounds中。
  public void getRootBounds(RectF bounds);
}
```

之所以要设置这个回调，是因为本篇中的Drawable是有层次的。如果B 是 A的子图层，那应用在A上的变换矩阵自然应该应用到B上，所以提供这个回调可以让B获取应用在A上的变换矩阵，从而正确地进行绘制。

**在本篇文章中出现的所有Drawable都实现了`TransformAwareDrawable `与`TransfromCallback`。** 在`ArrayDrawable`中的`getTransfrom`中可以看出它的工作机制（实际上除了个别自身有缩放的图层如`ScaleTypeDrawable`, `MatrixDrawable`外的实现都是想以下这段代码一样）：

```java
  @Override
  public void getTransform(Matrix transform) {
    //如果有父图层，则获取应用在父图层上的变换矩阵
    if (mTransformCallback != null) {
      mTransformCallback.getTransform(transform);
    } else {
    //如果没有父图层，就获取单位矩阵
      transform.reset();
    }
  }
```

在`ScaleTypeDrawable`和`MatrixDrawable`中会将自身的变换矩阵通过`Matrix.confat(Matrix m)`传给`transform`。如此一来就实现了变换矩阵向下传递的功能。

`VisibilityAwareDrawable`与`VisibilityCallback`搭配使用，它提供了在自身可见度改变的时候的通知函数（`onVisibilityChange(boolean visible)`）和在自身绘制时发生通知的回调(`onDraw()`)。仅仅`GenericDraweeHierarchy.RootDrawable`实现了它。

##6 类图

由于类中方法、变量过多，作者对其做了大量精简，仅用于参考设计层次。

![Class Diagram](http://desmondyao.com/image/fresco/class_diagram_drawables.PNG)


[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md
[7]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(7)%20-%20%E8%A7%A3%E7%A0%81.md

