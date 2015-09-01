#Fresco源码分析(5) - 文件缓存
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

接下来的三掌主要分析Fresco的三级缓存，可以说是接近Fresco最底层的内容之一了。如果有不对的地方，还请指正。

##Fresco缓存机制

首先介绍Fresco中的几种缓存及其工作机制，你可以通过官方文档[Image Pipeline介绍](http://fresco-cn.org/docs/intro-image-pipeline.html#_)及[缓存](http://fresco-cn.org/docs/caching.html#_)先有一个感性的理解。

再摘一个官方的图：

![Image Pipeline](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/imagepipeline.jpg)

##文件缓存架构





[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md

[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md

[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20%E6%96%87%E4%BB%B6%E7%BC%93%E5%AD%98.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier

[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
