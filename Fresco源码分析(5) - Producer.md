#Fresco源码分析(5) - Producer
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

在这一章的内容中，我们来分析Fresco中的Producer是怎么通过ImageRequest建立起数据请求通道的。

##Fresco缓存机制

首先介绍Fresco中的几种缓存及其工作机制，你可以通过官方文档[Image Pipeline介绍](http://fresco-cn.org/docs/intro-image-pipeline.html#_)及[缓存](http://fresco-cn.org/docs/caching.html#_)先有一个感性的理解。

官方文档的图：

![Image Pipeline](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/imagepipeline.jpg)

##几个关键概念

###EncodedImage

`EncodedImage`类里封装了未解码图片的所有字节码，包括图片数据、尺寸、旋转角度和缩放尺寸。它使用`PooledByteBuffer`存储字节码，可以直接通过`CloseableReference<PooledByteBuffer>`构造。

它有这两个主要函数：

- `getInputStream()` 返回存储着字节码的InputStream；
- `parseMetaData()` 解析字节码中的尺寸、旋转角度和缩放尺寸；

###CacheKey

Fresco中专门用于缓存键的接口，有这两种类实现了`CacheKey`：

- `BitmapMemoryCacheKey` 用于已解码的内存缓存键，会对Uri字符串、缩放尺寸、解码
- `SimpleCacheKey` 普通的缓存键实现，使用传入的HashCode作为唯一标识，所以需要保证相同键传入字符串相同。

##文件缓存




[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md


[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier
[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
