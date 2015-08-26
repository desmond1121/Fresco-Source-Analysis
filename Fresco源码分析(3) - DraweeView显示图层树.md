#Fresco源码分析(3) - DraweeView显示图层树
---

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

Fresco的源码中，DraweeView的介绍简洁明了：我就是把DraweeHierarchy显示到屏幕上的家伙。那我们来分析一下相关代码，看看它的逻辑是什么样的。

##时序图

首先可以用以下这个图初步理解SimpleDraweeView在调用了`setUri`之后的流程：

![DraweeView](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/setUri.PNG)

可以将这张图描述为以下信息：

- DraweeView直接显示DraweeHierarchy；
- DraweeController绑定数据源`DataSource`与数据订阅者`DataSubscriber`；

这张图还有部分内容没有画出：`DataSource`通过调用`notifyDataSubscriber`通知`DataScriber`更新`DraweeHierarchy`，从而更新视图。这部分内容和ImagePipeline部分交叉较多，将会在后续的章节中介绍，本章仅先分析DraweeView的绘制原理。

##基类DraweeView

`DraweeView<DH extends DraweeHierarchy>`是Fresco视图中的基类，使用的泛型必须是`DraweeHierarchy`，它继承了`ImageView`。

`DraweeView`内部的函数不多，**它们全部都是通过DraweeHolder的相应函数实现的。**主要有以下这几个：
- `void init(Context context)` 初始化`DraweeHolder`；
- `void setHierarchy(DH hierarchy)` 设置图层树，会调用 `super.setImageDrawable(mDraweeHolder.getTopLevelDrawable())`将图层树显示出来；
- `void setController(@Nullable DraweeController draweeController)` 设置`DraweeController`，像`setHierarchy`一样同样也会显示图层树；
- `Drawable getTopLevelDrawable()` 获取顶层图层Drawable；
- `View`中的`onAttachedToWindow()`、 `onDetachedFromWindow()`、 `onStartTemporaryDetach()`、 `onFinishTemporaryDetach()` 四个回调函数，提供当视图被绑定/解绑到指定布局上时的回调函数，它们会触发`DraweeHolder`的`onDetach()`或`onAttach()`；
- `void onTouchEvent(MotionEvent event)` 提供触屏反应。

在使用它之前要注意官方的API注释中有这么一段话：

>*你只能在将DraweeView仅仅当做ImageView来使用时才调用`setImageXXX`函数。*

要记住它：无论什么形式的`DraweeView`，目标图片的设置最终都是通过`DraweeController`。我们会在下一章中介绍`DraweeController`，它是连接`DraweeHierarchy`与`Image Pipeline`的桥梁。虽然`setHierarchy`与`setController`都会显示出图层树，但是实际上**`setHierarchy`在显示图片的时候只是把`DraweeView`当做普通的`ImageView`使用**，要使用Fresco的缓存、加载机制，必须使用`DraweeController`。

不过所幸Fresco实现了`SimpleDraweeVew`帮我们来处理这些繁琐的过程，它封装了`Controller`的使用。

##DraweeHolder

`DraweeHolder`充斥在`DraweeView`的各个位置，每个`DraweeView`的函数都是由它的对应函数执行的。它随着`DraweeView`的产生而初始化。在深入了解视图绘制之前，我们有必要了解它是做什么的。

它主要是用来维持`DraweeHierarchy`和`DraweeController`之间的沟通的。通过`create( DH hierarchy, Context context)`来创建实例，`DraweeHierarchy`通过第一个参数赋值，其中第二个参数用于`registerWithContext(Context)`（该函数暂时没有完善好）。

它有几个主要使用的函数：
- `void setHierarchy(DH hierarchy)` 在`DraweeView.setHierarchy`中被调用，将DraweeHierarchy传给持有的DraweeController；
- `void setController(DraweeController draweeController)` 在`DraweeeView.setController`中被调用无条件解绑旧的Controller（如果存在的话），并将旧DraweeController的DraweeHierarchy设置为null，并调用`DraweeController.onDetach()`将它变为解除绑定状态；将持有的DraweeHierarchy（如果有的话）赋给新传入的DraweeController并调用`DraweeController.onAttach()`让它变为绑定状态。（**更换DraweeController确是一个非常耗时的过程，应该尽量避免为视图指定新的DraweeController，参考[官方文档](http://fresco-cn.org/docs/using-controllerbuilder.html#draweecontroller)）**
- `void attachController()` 若所属的`DraweeView`未绑定`DraweeController`，`DraweeController`成员变量不为空并且已经设置过图层树之后，调用该成员变量的`onAttach`方法；
- `void detachController()` 若所属的`DraweeView`已绑定`DraweeController`，`DraweeController`成员变量不为空，调用该成员变量的`onDetach`方法；
- `void attachOrDetachController()` 当已绑定`DraweeController`并且所属的`DraweeView`可见时，调用`attachController`；否则调用`detachController`。

特别地，`DraweeHolder`继承了`VisibilityCallback`，为`DraweeHierarchy.RootDrawable`提供回调：当图层的Visibility属性改变的时候对`DraweeController`调用`attachOrDetachController`操作，当图层不可见时释放资源。

##GenericDraweeView

`GenericDraweeView`就是使用`GenericDraweeHierarchy`图层树的视图，它承担了所有的xml属性交互工作。

*实际上在图层树上目前Fresco也只实现了一个`GenericDraweeHierarchy`，使用泛型是为了后续的开发便利。*

它会在初始化的时候调用`inflateHierarchy(Context context, AttributeSet attrs)`函数，从xml的中获取如下几个属性（如果存在的话）：
- `fadeDuation` 渐隐/渐显动画时间；
- `aspectRatio` 图片长宽比例，参考[官方文档](http://fresco-cn.org/docs/using-drawees-xml.html#wrap-content)；
- ``XXXImage`/`XXXImageScaleType` 各图层要显示的Drawable（除了目标显示图层）及它们的`ScaleType`；
- `RoundingParams`中的参数

在它的Measure过程中，会依次判断是否高度、宽度属性中有`wrap_content`，会将先判断到的属性更正为实际长度。**但是Fresco并不支持使用wrap_content。**如果你非要使用，只能使用一边，然后搭配`aspectRatio`使用。

**在GenericDraweeView获取完xml属性之后，它会通过`GenericDraweeHierarchyBuilder.build`创造一个与之对应的GenericDraweeHierarchy作为默认使用的图层树，并调用`setHierarchy`方法将其传递给`DraweeHolder`并显示出来。**

##SimpleDraweeView

在SimpleDraweeView中的函数就更少了，可以明确的说，它就只是将GenericDraweeHierarchy显示到UI界面上的空间而已。它比GenericDraweeView多出来的功能就是**它内部提供了最简单的DraweeController实现**。它有两个函数比较需要关注：

- `initialize`函数，这个函数会在`Fresco.initialize`中调用，目的就是初始化DraweeControllerBuilder，用于构建DraweeController。

- `setImageURI` 这个函数我们会经常调用，因此有必要看看它的实现和普通ImageView的`setImageXXX`方法有什么区别：

```java
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller = mSimpleDraweeControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
    setController(controller);
  }
```

而`setController`会调用`DraweeHolder.setController`，将图层树的控制权交给DraweeController，并显示出图层树。

##类图

![DraweeView Diagram](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/class_diagram_3.PNG)

