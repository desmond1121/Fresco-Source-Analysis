#Fresco源码解读(1) - Drawable & Hierarchy
---

@(Android技术)[源码阅读|Fresco]

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

##1. 主要Drawable类

首先介绍几种Fresco中实现几个的Drawable，了解它们会帮助你理解Fresco加载图像的原理。它们都直接或间接继承了`Drawable`，但是各自的功能是不一样的。经过总结，我认为其中一共有三种功能的Drawable：层次型、容器型和视图型。

###引论：给图像分层次是什么作用？

如果你使用过Fresco这个强大的库之后，你就知道它可以在一个图像的加载、绘制过程中实现极大的定制化。你可以设置进度条来显示图片加载/下载的进度，可以设置占位图等到图片加载/下载成功后再显示目标图片，可以让在加载/下载失败后显示失败图片（更多功能参考[Fresco中文文档](http://fresco-cn.org/docs/)）。Fresco将进度条、占位图、失败图都作为图像的一层视图来管理，让负责视图功能部分与逻辑层尽可能实现解耦。

直接上源码的一个视图例子就好理解了，作者进行了适当修改与翻译。

 o 层次型Drawable（维持图层）
 
 |
 
 ------ 容器型Drawable（可对内容进行缩放）
 
 |　　　　|
 
 |　　　　--- 视图型Drawable（存放占位图）
 
 |
 
 ------ 容器型Drawable（可对内容进行缩放）
 
 |　　　　|
 
 |　　　　----- 容器型Drawable（可多次设置内容）
 
 |　　　　　　　|
 
 |　　　　　　　--- 视图型Drawable（存放目标图片）
 
 |
 
 ------ 容器型Drawable（可对内容进行缩放）
 
 |　　　　|
 
 |　　　　--- 视图型Drawable（存放重试图片）
 
 |
 
 ------ 容器型Drawable（可对内容进行缩放）
 
  　　　　|
  
  　　　　--- 视图型Drawable（存放失败图片）

（该例位于`com.facebook.drawee.generic.GenericDraweeHierarchy`的类注释中）

这个例子充分描述了一个图像的层次，当然也可以在设置的时候往里面自行设置所需要的图层。

###层次型Drawable

在这一节中介绍的Drawable并不直接负责具体图像绘制，而是负责组建图像层次。

####ArrayDrawable

`ArrayDrawable`内部存储着一个Drawable数组，工作原理与Android内置的`LayerDrawable`极其相似，我们可以看它的绘制函数：

```java
  @Override
  public void draw(Canvas canvas) {
    for (int i = 0; i < mLayers.length; i++) {
      mLayers[i].draw(canvas);
    }
  }
```

可见它将数组中的`Drawable`当做它的图层，在绘制的时候`ArrayDrawable`会按照数组顺序绘制其中的图层，数组最后的成员会显示在最上方。它与`LayerDrawable`不同的点就在于它**不支持动态的添加/删除图层，只能在初始化时通过传入的数组决定图层数**。不过好在它能够为存在的图层更换Drawable。（关于`LayerDrawable`可以参考我翻译的一文章：[Android LayerDrawable](http://blog.csdn.net/desmondj/article/details/47751553)。）

####FadeDrawable

`FadeDrawable`继承了`ArrayDrawable`。它除了具有`ArrayDrawable`本身的功能之外，还提供隐藏/显示图层的功能（可设置渐变）。具体的几个核心函数有：
 * `setTransitionDuration(int durationMs)` 设置隐藏/显示图层渐变动画时间（默认为300ms）。
 * `fadeInLayer(int index)` 显示指定图层
 * `fadeOutLayer(int index)` 隐藏指定图层
 * `fadeInAllLayers()`  显示所有图层
 * `fadeOutAllLayers()`  隐藏所有图层
 * `fadeToLayer(int index)` 显示指定图层同时隐藏其他图层
 * `fadeUpToLayer(int index)` 隐藏数组下标<=index的图层
 
它内部维护着一个boolean数组来维持需要显示的图层（可以调用`isLayerOn(int inxex)`查看指定图层是否显示）。
 
 
###容器型Drawable
 
**FowardingDrawable**

`FowardingDrawable`通俗的来说就是图片容器。它内部维护一个`Drawable`变量`mCurrentDelegate`，将Drawable的基本函数以及一些回调函数传递给目标图片，并在`draw(Canvas)`函数中调用`mCurrentDeletate.draw(Canvas)`函数将目标图片绘制出来。这个类本身仅仅起到包装作用，乍一看没什么了不起的，但是可以通过继承它并对图片处理做一些封装（如缩放、变形等）来达到一些想要的效果。

主要有这几种类继承`FowardingDrawable`：

- `ScaleTypeDrawable`：封装了对代理图片的缩放处理，具体参数与[Android ScaleType](http://developer.android.com/reference/android/widget/ImageView.ScaleType.html)相似。



##2. DraweeHierarchy

由于我们可能在视图中绘制与目标图片有关的其他图片（占位图、缩略图、圆角图等），那么维持一个视图的绘制层次就是一件比较必要的事情了，这么做能够帮我们控制绘制过程中图片的显示顺序。

**DraweeHierarchy**是一个保持视图层次的组件。相比于Android内置的视图层次逻辑，它显得更加轻量级。所有的Hierarchy都继承`DraweeHierarchy`接口，这个接口中仅仅提供了获取视图最顶层Drawable的函数：

```java
public interface DraweeHierarchy {

  public Drawable getTopLevelDrawable();
}
```

内置的Hierarchy只有一个`GenericDraweeHierarchy`，它是通过继承`SettableDraweeHierarchy`实现的。

它内部可以保存视图层次的几种实例：

- PlaceHolderImage 占位图
- ProgressBarImage 进度条图
- 



##参考文献：

1. [Fresco中文文档](http://fresco-cn.org/docs/)



