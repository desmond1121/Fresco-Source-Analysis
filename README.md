#Fresco-Source-Analysis
中文的Fresco源码解读，分析版本：0.7.0。欢迎Issue讨论！

## 导读


本系列文章目的是分析源码，具体使用方法虽然会涉及，但不多。你应该在对Fresco的使用方法有一定了解之后再来阅读这些文章，推荐[Fresco中文文档](http://fresco-cn.org/docs/)来了解它的使用方法。

### Fresco中几个基本设计模式

在[Wiki-设计模式](https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F)中介绍了Fresco中用到的几个基本设计模式，供读者参考。

### 初级分析：DraweeView显示图层树的过程

这部分的内容针对使用者会接触到的几个部件：DraweeHierarchy、DraweeController、DraweeView、ImageRequest等，介绍这些组件的功能，分析它们之间的关系。具体参考以下章节：

- [Fresco源码分析(1) - 图像层次与各类Drawable][1]
- [Fresco源码分析(2) - GenericDraweeHierarchy构建图层][2]
- [Fresco源码分析(3) - DraweeView显示图层树][3]

### 进阶分析：图像请求与缓存

这部分的内容从DataSource的功能出发，探索与分析Fresco底层缓存、加载数据的原理实现。具体参考以下章节：

- [Fresco源码分析(4) - 异步加载数据][4]
- [Fresco源码分析(5) - Producer流水线][5]
- [Fresco源码分析(6) - 缓存][6]

### 待完成

- Fresco源码分析(7) - 解码

Fresco的解码能力是它的一大特色，它会根据不同平台放置解析完成的图片，根据不同的渠道（网络、本地）处理渐进式JPEG图。由于作者的能力与时间有限，在图像解码方面的知识储备不足，解码部分暂未完成，有能力的朋友可以联系我一起进行分析，谢谢！

###总结

作者总结了Fresco的架构图如下：

![Fresco](http://desmondyao.com/image/fresco/fresco.PNG)

其中：

- **表现层** 维持图片层次DraweeHierarchy并将其绘制到屏幕上；
- **控制层** 提供缓存查找、图片获取及处理的接口，通过DraweeController来控制表现层的DraweeHierarchy，它是表现层与核心层之间的纽带；
- **核心层** 负责各类缓存、解码、数据获取的逻辑，将数据通过Producer传达给控制层的Consumer。

[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md

如有错误，希望指正，谢谢！

Authored by Desmond
