#Fresco-Source-Analysis
中文的Fresco源码解读，持续更新中。Fresco source code analysis in Chinese, still updating.

## 导读

### Fresco中几个基本设计模式

为了不在后面文档中赘述和加强读者理解，在[Wiki-设计模式](https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F)中介绍了Fresco中用到的几个基本设计模式。

### 初级分析：DraweeView显示图层树的过程

这部分的内容针对使用者会接触到的几个部件：DraweeHierarchy、DraweeController、DraweeView、ImageRequest等，介绍这些组件的功能，分析它们之间的关系。

总结`SimpleDraweeView`调用`setUri(Uri uri)`后的调用图：
![DraweeView](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/sequence_diagram_setUriSeq1.PNG)

（`DataSource`在进阶分析中：[Fresco源码分析(4) - 异步加载数据](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md)）

具体参考以下章节：
- [Fresco源码分析(1) - 图像层次与各类Drawable](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md)
- [Fresco源码分析(2) - GenericDraweeHierarchy构建图层](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md)
- [Fresco源码分析(3) - DraweeView显示图层树](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md)

### 进阶分析：Fresco的图像请求与缓存

这部分的内容从DataSource的功能出发，探索与分析Fresco底层缓存、加载数据的原理实现。

根据Uri生成`Producer`和`DataSource`的过程概览：
![DataSource](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/sequence_diagram_setUriSeq2.PNG)

具体参考以下章节：
- [Fresco源码分析(4) - 异步加载数据](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md)
- [Fresco源码分析(5) - Producer与缓存](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E4%B8%8E%E7%BC%93%E5%AD%98.md) 未完成


如有错误，希望指正，谢谢！

Authored by Desmond
