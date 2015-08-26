#Fresco源码分析(4) - ImagePipeline之关键概念介绍
---

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

这一章中我们首先介绍并简单分析几个关键概念，不然很可能会对后续的理解一头雾水。

##ImagePipeline介绍

内容可以直接参照[ImagePipeline介绍](http://fresco-cn.org/docs/intro-image-pipeline.html)

##可关闭的引用

Facebook在Java中实现了具有引用计数功能的类：`SharedReference<T>`（注意不是Android里的SharedPreference）。它将类型为`T`的对象进行包装，为其实现增加引用数、减少引用数、删除引用的功能。

特别地，它会使用专门用于实现释放资源的接口：`ResourceReleaser<T>`，它内部只有一个函数：`void release(T object)`。

当然，但是直接让程序员直接去操作它很可能会出现问题。所以`CloseableReference`出现了，它为任何实现了`Closeable`类的对象封装了引用计数、回收引用的功能。

`CloseableReference`中有几个主要函数：

- `CloseableReference<T> of(T object, ResourceReleaser<T> releaser)` **用于初始化引用计数（而不是使用构造函数）**，该函数会新建一个`SharedReference`将对象包装起来，引用计数为1。若不传releaser，会使用默认的`ResourceReleaser`，调用`Closeable.close()`函数回收引用；
- `CloseableReference<T> clone()` 用于添加对象引用，该函数会新建一个`CloseableReference`，同时持有的`SharedReference`引用+1， **不会创建`SharedReference`对象**；
- `T get()` 返回引用对象；
- `void close()` 减少一个引用计数，若引用计数减为0，则调用releaser的`release(T object)`将对象释放；

*注意`CloseableReference`不会在引用计数减少到0的时候自动释放对象，这也是为什么必须调用`close()`的原因。* 你需要谨慎地使用这个工具，使用方法参考[Fresco中文文档](http://fresco-cn.org/docs/closeable-references.html#_)

【Fresco内的各类Closeable对象待完成】

##ImageRequest

`ImageRequest`存储着Image Pipeline处理被请求图片所需要的信息。 **它仅仅用来装信息，而且一经初始化后就无法改变内容（即Immutable，不过可以获取内容）。**它的初始化只能通过`fromUri(Uri uri)`或`ImageRequestBuilder.build()`来实现。

我们来看看它内部存储着一些什么信息：

- `ImageType`：若为`ImageType.SMALL`，则所请求的图片会存储在专用小文件缓存中。具体参考[Fresco缓存](http://fresco-cn.org/docs/caching.html)；
- `SourceUri`：图片源Uri；
- `SourceFile`：图片文件地址（若是本地文件的话）；
- `ProgressiveRenderingEnabled` 若为true，则这个图片请求会返回质量递进的几次图片信息（渐进式图片）；
- `LocalThumbnailPreviewsEnabled` 若为true，则这个图片请求会在访问本地图片时先返回一个缩略图；
- `ResizeOptions` 缩放尺寸，仅支持JPEG，而且不是每次都需要在ImageRequest中设置缩放尺寸的，具体使用请参照[Fresco缩放和旋转图片](http://fresco-cn.org/docs/resizing-rotating.html#)；
- `AutoRotateEnabled` 是否允许图片旋转，具体参照[Fresco缩放和旋转图片](http://fresco-cn.org/docs/resizing-rotating.html#)；
- `RequestPriority` 这个请求的优先级：
- `LowestPermittedRequestLevel` 最低允许从哪层缓存中取数据，参考[最低请求级别](http://fresco-cn.org/docs/image-requests.html#)；
- `IsDiskCacheEnabled` 若为false，则此图片请求不会从文件缓存中获取数据；
- `PostProcessor` 在图片请求成功之后对图片的处理操作，参考[修改图片](http://fresco-cn.org/docs/modifying-image.html#_)。



参考文献：

[FaceBook推出的Android图片加载库-Fresco](http://blog.csdn.net/bboyfeiyu/article/details/44943959)