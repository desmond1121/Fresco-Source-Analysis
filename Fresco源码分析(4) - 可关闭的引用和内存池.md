#Fresco源码分析(4) - 可关闭的引用和内存池
---

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

这一章中我们首先介绍并简单分析几个关键概念，不然很可能会对后续的理解一头雾水。

##ImagePipeline介绍

Image Pipeline是Fresco中负责加载数据的工具，它的功能和缓存分级可以直接参照[ImagePipeline介绍](http://fresco-cn.org/docs/intro-image-pipeline.html)

##可关闭的引用

Facebook在Java中实现了具有引用计数功能的类：`SharedReference<T>`（注意不是Android里的SharedPreference）。它将类型为`T`的对象进行包装，为其实现增加引用数、减少引用数、删除引用的功能。

特别地，它会使用专门用于实现释放资源的接口：`ResourceReleaser<T>`，它内部只有一个函数：`void release(T object)`。

当然，但是直接让程序员直接去操作它很可能会出现问题。所以`CloseableReference`出现了，它为任何实现了`Closeable`类的对象封装了引用计数、回收引用的功能。

`CloseableReference`中有几个主要函数：

- `CloseableReference<T> of(T object, ResourceReleaser<T> releaser)` **用于初始化引用计数（而不是使用构造函数）**，该函数会新建一个`SharedReference`将对象包装起来，引用计数为1。若不传releaser，会使用默认的`ResourceReleaser`，调用`Closeable.close()`函数回收`Closeable`引用；
- `CloseableReference<T> clone()` 用于添加对象引用，该函数会新建一个`CloseableReference`，同时持有的`SharedReference`引用+1， **不会创建`SharedReference`对象**；
- `T get()` 返回引用对象；
- `void close()` 减少一个引用计数，若引用计数减为0，则调用releaser的`release(T object)`将对象释放。 **一旦创建了一个CloseableReference，当持有者离开作用域时就必须调用这个函数！**

你需要谨慎地使用这个工具，使用方法参考[Fresco中文文档](http://fresco-cn.org/docs/closeable-references.html#_)

###CloseableImage

Fresco中在很多地方使用了CloseableReference，首先介绍两种应用在图像上的：

- CloseableStaticBitmap 它内部持有一个`CloseableReference<Bitmap>`包装目标`Bitamp`，以及关于图像质量、旋转角度的信息。在`close`的时候调用`CloseableReference`的`close()`函数释放资源。
- CloseableAnimatedBitmap 它内部持有一个`List<CloseableReference<Bitmap>>`包装起每一帧的`Bitmap`，还存有每一帧的时长。

###类图



##内存池



参考文献：

[FaceBook推出的Android图片加载库-Fresco](http://blog.csdn.net/bboyfeiyu/article/details/44943959)