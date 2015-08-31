#Fresco源码分析(4) - 异步加载数据
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

在这一章中，我将分析Fresco是怎么根据Uri生成`DataSource`与`Producer`的（关于Producer见[Wiki](https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer)）。

##1 时序图

根据Uri生成`Producer`和`DataSource`的过程概览：
![DataSource](http://desmondtu.oss-cn-shanghai.aliyuncs.com/Fresco/sequence_diagram_setUriSeq2.PNG)

以上的时序图是精简过的，意在表达各个组件之间的调用顺序和功能。

##2 通过Url获取DataSourceSupplier

在[第三章末尾][3]我们讲到`DraweeControllerBuilder`会根据Uri初始化`DataSource`的[Supplier][Supplier]，那么现在来看看是怎么初始化的。

首先会根据各种情况判断生成不同的DataSource：

    protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier() {

        //...

        if (mImageRequest != null) {
          supplier = getDataSourceSupplierForRequest(mImageRequest);
        } else if (mMultiImageRequests != null) {
          supplier = getFirstAvailableDataSourceSupplier(mMultiImageRequests);
        }
我们看到Fresco会根据是否有多个图片请求，如果有，则会取多个图片请求中最先能请求到的作为DataSource。

        if (supplier != null && mLowResImageRequest != null) {
          List<Supplier<DataSource<IMAGE>>> suppliers = new ArrayList<>(2);
          suppliers.add(supplier);
          suppliers.add(getDataSourceSupplierForRequest(mLowResImageRequest));
          supplier = IncreasingQualityDataSourceSupplier.create(suppliers);
        }
在之后会判断是否有提供低分辨率的图片，如果有，则创建一个清晰度逐渐提高的DataSource。更多关于多图请求的内容参照[多图请求及图片复用](http://fresco-cn.org/docs/requesting-multiple-images.html#)。

在此我们就分析根据一个ImageRequest获取DataSource的过程，获取其他种类DataSource都是通过它实现的。

    protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      boolean bitmapCacheOnly) {
        if (bitmapCacheOnly) {
          return mImagePipeline.fetchImageFromBitmapCache(imageRequest, callerContext);
        } else {
          return mImagePipeline.fetchDecodedImage(imageRequest, callerContext);
        }
    }

这个函数给我们传递了两个信息：

- **Fresco中数据的传递都是包装在CloseableReference<CloseableImage>中的**！
- 它会根据`bitmapCacheOnly`（是否只从已解码的内存缓存中获取数据）来获取不同的DataSource。

那我们就来看看它不限制缓存级别的时候是怎么处理的（这种情况下会包括`bitmapCacheOnly`时候的内容）。

    public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext) {
        try {
          Producer<CloseableReference<CloseableImage>> producerSequence =
              mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
          return submitFetchRequest(
              producerSequence,
              imageRequest,
              ImageRequest.RequestLevel.FULL_FETCH,
              callerContext);
        } catch (Exception exception) {
          return DataSources.immediateFailedDataSource(exception);
        }
    }

我们发现它会首先根据传入的`ImageRequest`产生一个`Producer`，


[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md

[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier

[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
