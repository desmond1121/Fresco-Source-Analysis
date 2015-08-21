#Fresco源码分析(3) - DraweeView显示图层树
---

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

Fresco的源码中，DraweeView的介绍简洁明了：我就是把DraweeHierarchy显示到屏幕上的家伙。那我们来分析一下相关代码，看看它的逻辑是什么样的。

##基类DraweeView

`DraweeView<DH extends DraweeHierarchy>`是Fresco视图中的基类，使用的泛型必须是`DraweeHierarchy`。它继承了`ImageView`，在使用它之前要注意官方的API注释中有这么一段话：

>*你只能在将DraweeView仅仅当做ImageView来使用时才调用`setImageXXX`函数。*

要记住它：无论什么形式的`DraweeView`，目标图片的设置最终都是通过`DraweeController`。我们会在下一章中介绍`DraweeController`，它是连接`DraweeHierarchy`与`Image Pipeline`的桥梁。

`DraweeView`内部的函数不多，**它们全部都是通过DraweeHolder相应函数实现的。**主要有以下这几个：
- `void setHierarchy(DH hierarchy)` 设置图层树，会调用
- `void setController(@Nullable DraweeController draweeController)` 设置`DraweeController`
- `Drawable getTopLevelDrawable()` 获取顶层图层Drawable；
- `View`中的`onAttachedToWindow()`、 `onDetachedFromWindow()`、 `onStartTemporaryDetach()`、 `onFinishTemporaryDetach()` 四个回调函数，提供当视图被绑定/解绑到指定布局上时的回调函数，它们会触发`DraweeHolder`的`onDetach()`或`onAttach()`。
- `onTouchEvent(MotionEvent event)` 提供触摸反应


##GenericDraweeView

`GenericDraweeView`就是使用`GenericDraweeHierarchy`图层树的视图，它承担了所有的xml属性交互工作。

*ps: 实际上在图层树上目前Fresco也只实现了一个`GenericDraweeHierarchy`，使用泛型是为了后续的开发便利。*






