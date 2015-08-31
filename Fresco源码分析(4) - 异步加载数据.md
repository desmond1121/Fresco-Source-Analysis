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
在之后会判断是否有提供低分辨率的图片，如果有，则创建一个清晰度逐渐提高的DataSource（默认只有两级清晰度）。更多关于多图请求的内容参照[多图请求及图片复用](http://fresco-cn.org/docs/requesting-multiple-images.html#)。

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

我们发现它会首先根据传入的`ImageRequest`产生一个`Producer`，具体过程会在[Fresco源码分析(5) - Producer与缓存][5]中分析，现在你只需要知道 **Producer提供了数据传送的管道**。

在`submitFetchRequest`函数中做了三件事：

1. 取`ImageRequest`的`LowestPermittedRequestLevel`和传入的`RequestLevel`中最高的一级作为此次数据获取的最高缓存获取层；
2. 将`ImageRequest`、本次请求的唯一标识、`ImageRequestListener`（提供ImageRqeuest事件的回调）、是否需要渐进式加载图片等信息封装进`SettableProducerContext`。
3. 创建`AbstractproducerToDataSourceAdapter`，它实际上是一种`DataSource`，在这个过程中会让producer通过`SettableProducerContext`获取数据。

至此我们就获取了所需要的DataSource，并将它设置给DraweeController。

##3 各类DataSource

上一节中我们获取了一个`AbstractproducerToDataSourceAdapter`作为`DataSource`，你可能对它的功能不太理解，这一节将介绍各类DataSource的功能。

###3.1 基类

DataSource的原型：

    public interface DataSource<T> {

        public boolean isClosed();

        //是否得到结果
        boolean hasResult();

        //获取现在得到的结果
        @Nullable T getResult();

        //是否结束
        boolean isFinished();

        //是否失败
        boolean hasFailed();

        //获取失败原因
        @Nullable Throwable getFailureCause();

        //获取加载进度
        float getProgress();

        //释放资源
        boolean close();

        //绑定DataSubScriber
        void subscribe(DataSubscriber<T> dataSubscriber, Executor executor);
    }

各个函数的功能已经在上面介绍得很清楚了，Fresco接着实现了一个基础类`AbstractDataSource`，维持着IN_PROGRESS、SUCCESS、FAILURE三种状态。它除了实现`close`、`subscribe`的之外，添加了这几个重要函数及其实现：

- `setResultInternal(@Nullable T value, boolean isLast)` 当DataSource关闭或状态非IN_PROGRESS时返回false，否则设置Result并返回ture，若isLast为true，则设置状态为SUCCESS；
- `setFailureInternal(Throwable throwable)` 当DataSource关闭或状态非IN_PROGRESS时返回false，否则设置失败原因、状态为FAILURE并返回true；
- `setProgressInternal(float progress)` 当DataSource关闭或状态非IN_PROGRESS时返回false，若目标进度小于已有进度也返回false，否则更新进度并返回true；
- `notifyDataSubscriber(DataSubscriber<T> subscriber, Executor executor, boolean isFailure, boolean isCancellation)` 在指定的Executor上根据isFailure、isCancellation调用DataSubscriber的对应函数（若都为false即成功加载），由此更新数据；
- `notifyDataSubscribers()` 通知所有绑定的DataSubscriber更新数据；
- `setResult`/`setFailure`/`setProgress` 调用对应`setXXXInternal`，当其返回true时，调用`notifyDataSubscribers()`。

###3.2 各类DataSource及其功能介绍

###3.2.1 AbstractProducerToDataSourceAdapter

`AbstractProducerToDataSourceAdapter`包装了Producer取数据的过程。

**CloseableProducerToDataSourceAdapter**

**ProducerToDataSourceAdapter**

###3.2.2 其他DataSource

**IncreasingQualityDataSource**

它内部维持着一个`AbstractDataSource`的列表，**DataSource提供数据的清晰度由后往前递增**，所以每次获取新DataSource的时候必须是在之前获取的DataSource之前，并且会将上次使用的DataSource销毁。

**FirstAvailableDataSource**

**SettableDataSource**

它继承AbstractDataSource，并将重写`settResult`、`setFailure`、`setProgress`在内部调用父类的相应函数，**但是修饰符变成了public**（原来是protected）。即使用`SettableDataSource`时可以在外部调用这三个函数设置DataSource状态。一般用于在获取DataSource失败时直接产生一个设置为Failure的DataSource。

###4 Producer



[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md

[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md

[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier

[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
