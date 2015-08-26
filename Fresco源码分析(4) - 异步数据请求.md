#Fresco源码分析(4) - 异步数据请求
---

> 作者：[Desmond 转载请注明出处！](http://blog.csdn.net/desmondj)

这一章中我们首先分析Fresco的数据请求过程，由此着手分析Fresco的Image Pipeline。

##引论

首先上一张[Fresco中文文档](http://fresco-cn.org/docs/intro-image-pipeline.html#_)中的图：

![Image](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/imagepipeline.jpg)

以及在[Fresco中文文档](http://fresco-cn.org/docs/intro-image-pipeline.html#_)中对Fresco的加载路程的说明：92

1. 检查内存缓存，如有，返回
2. 后台线程开始后续工作
3. 检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。
4. 检查是否在文件缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。
5. 从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中。

##ImageRequest

`ImageRequest`存储着Image Pipeline处理被请求图片所需要的信息。它一经初始化后就无法改变内容（即Immutable，不过可以获取内容）的对象。它的初始化只能通过`fromUri(Uri uri)`或`ImageRequestBuilder.build()`来实现。

我们来看看它内部存储着一些什么信息：

- `ImageType`：若为`ImageType.SMALL`，则所请求的图片会存储在专用小文件缓存中。具体参考[Fresco缓存](http://fresco-cn.org/docs/caching.html)
- `SourceUri`：图片源Uri
- `SourceFile`：图片文件地址（若是本地文件的话）
- `ProgressiveRenderingEnabled` 若为true，则这个图片请求会返回质量递进的几次图片信息（渐进式图片）
- `LocalThumbnailPreviewsEnabled` 若为true，则这个图片请求会在访问本地图片时先返回一个缩略图。
-


参考文献：

[FaceBook推出的Android图片加载库-Fresco](http://blog.csdn.net/bboyfeiyu/article/details/44943959)