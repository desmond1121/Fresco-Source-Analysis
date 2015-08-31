#Fresco-Source-Analysis
中文的Fresco源码解读，持续更新中。Fresco source code analysis in Chinese, still updating.

## 导读

### 初级分析：DraweeView显示图层树的过程

总结`SimpleDraweeView`调用`setUri(Uri uri)`后的调用图：
![DraweeView](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/sequence_diagram_setUri.PNG)

`DataSource`会在数据加载好后对图层进行修改，具体见[Fresco源码分析(4) - 异步加载数据](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md)中分析它的工作机制）

具体参考下面三个章节：
- [Fresco源码分析(1) - 图像层次与各类Drawable](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md)
- [Fresco源码分析(2) - GenericDraweeHierarchy构建图层](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md)
- [Fresco源码分析(3) - DraweeView显示图层树](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md)

### 进阶分析：Fresco的图像请求与缓存

- [Fresco源码分析(4) - 异步加载数据](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md)


Authored by Desmond
