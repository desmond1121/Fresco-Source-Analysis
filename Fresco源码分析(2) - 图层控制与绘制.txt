#Fresco源码分析(2) - 图层控制与绘制

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj) 

在本篇的内容中，作者将初步介绍Fresco是怎么将Drawable层次绘制到`DraweeView`中的。

##图层控制逻辑

###引论

`DraweeHierarchy`是所有`Hierarchy`的父接口，它内部只提供了一个基本而又不可缺失的功能：获取图层树的父节点图层。不过仅仅只有这个功能是不够的，Fresco紧接着用接口`SettableHierarchy`来继承它，声明一些具体的功能：

- `void reset()` 重置所有图层（慎用）
- `void setImage(Drawable drawable, float progress, boolean immediate)` 设置目标显示图片，`progress`表示图片的加载质量（在渐进式图中使用），`immediate`设置为true时图片会马上显示（而不是有一个渐变的过程）。
- `void setProgress(float progress, boolean immediate)`  设置`ProgressBar`的进度。当`progress`为1时会隐藏`ProgressBar`，同样`immediate`设置为true时进度条会马上消失。
- `void setFailure(Throwable throwable)` 图片加载失败的回调，可以用来显示失败提示图片。参数为加载失败时抛出的异常。
- `void setRetry(Throwable throwable)` 图片加载失败但是还希望再次尝试加载时的回调，可以用来显示重试提示图片。参数为加载失败时抛出的异常。
- `void setControllerOverlay(Drawable drawable)` 由于特殊原因要用指定图层覆盖已有图层时的回调，传入的参数会显示在`ArrayDrawable`的最上方。

在接下来的内容中会介绍本节的主角：**GenericDraweeHierarchy**。它实现了`SettableHierarchy`接口，你可以从这个类中看到大部分Fresco处理图层的逻辑。

###GenericDraweeHierarchy



***需要注意的几点：***
-  一个Drawable实例只能与一个DraweeHierarchy绑定！如果绑定了多个DraweeHierarchy，会出问题。
-  如果你不显式地设置这些图层，它们是不会被创造的，**除了占位图层之外**，它会默认创造一个透明图层。
-  如果设置了背景图层，它会一直显示。


