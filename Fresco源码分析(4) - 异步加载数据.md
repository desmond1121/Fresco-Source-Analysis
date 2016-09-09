#Fresco源码分析(4) - 异步加载数据
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

在这一章中，我将分析数据源`DataSource`的生成过程及它的作用。

##1 时序图

根据Uri生成`Producer`和`DataSource`的过程概览：
![DataSource](http://desmondyao.com/image/fresco/sequence_diagram_setUriSeq2.PNG)

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

- **Fresco中数据的传递基本都是包装在[CloseableReference][Closeable]中的**！
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

我们发现它会**首先根据传入的`ImageRequest`产生一个`Producer`，它为不同ImageRequest提供了不同的数据传送的管道**。（关于Producer具体见[Fresco源码分析(5) - Producer][5]）

在`submitFetchRequest`函数中做了三件事：

1. 取`ImageRequest`的`LowestPermittedRequestLevel`和传入的`RequestLevel`中最高的一级作为此次数据获取的最高缓存获取层；
2. 将`ImageRequest`、本次请求的唯一标识、`ImageRequestListener`（提供ImageRqeuest事件的回调）、是否需要渐进式加载图片等信息封装进`SettableProducerContext`。
3. 创建`AbstractproducerToDataSourceAdapter`，它实际上是一种`DataSource`，在这个过程中会让producer通过`SettableProducerContext`获取数据。

至此我们就获取了所需要的DataSource，并将它设置给DraweeController。

##3 各类DataSource

上一节中我们获取了一个`AbstractproducerToDataSourceAdapter`作为`DataSource`，你可能对它的功能不太理解，这一节将介绍各类DataSource的功能。

###3.1 基类

DataSource的原型

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

Fresco接着实现了一个基础类`AbstractDataSource`，维持着IN_PROGRESS、SUCCESS、FAILURE三种状态。它除了实现`close`、`subscribe`的之外，添加了这几个重要函数及其实现：

- `setResultInternal(@Nullable T value, boolean isLast)` 当DataSource关闭或状态非IN_PROGRESS时返回false，否则设置Result并返回ture，若isLast为true，则设置状态为SUCCESS；
- `setFailureInternal(Throwable throwable)` 当DataSource关闭或状态非IN_PROGRESS时返回false，否则设置失败原因、状态为FAILURE并返回true；
- `setProgressInternal(float progress)` 当DataSource关闭或状态非IN_PROGRESS时返回false，若目标进度小于已有进度也返回false，否则更新进度并返回true；
- `notifyDataSubscriber(DataSubscriber<T> subscriber, Executor executor, boolean isFailure, boolean isCancellation)` 在指定的Executor上根据isFailure、isCancellation调用DataSubscriber的对应函数（若都为false即成功加载），由此更新数据；
- `notifyDataSubscribers()` 通知所有绑定的DataSubscriber更新数据；
- `setResult`/`setFailure`/`setProgress` 调用对应`setXXXInternal`，当其返回true时，调用`notifyDataSubscribers()`。

###3.2 AbstractProducerToDataSourceAdapter

`AbstractProducerToDataSourceAdapter`也是一种DataSource，它继承`AbstractDataSource`，包装了Producer取数据的过程，非常重要，所以单独列出来。

它在创造过程中主要做了两件事：

1. 创建一个Consumer，在newResult、Failure、Cancel、ProgressUpdate几个状态函数中调用自己的相应实现；
2. 对之前根据ImageRequest创造出来的的Producer调用`produceResults(Consumer consumer, ProducerContext context)`，让它开始加载数据。

我们用一个直观的图来看看当Producer产生新结果(newResult)时的调用顺序是什么样的：

![NewResult](http://desmondyao.com/image/fresco/NewResultExample.PNG)

当失败(Failure)、取消(Cancel)、更新进度(ProgressUpdate)时的操作流程是类似的。我们可以看出，**`AbstractProducerToDataSourceAdapter`是连接Producer与DataSource的纽带。**

**AbstractProducerToDataSourceAdapter的构造方法是protected，只能通过继承的类提供的办法来构造它**。以下是两个继承了`AbstractProducerToDataSourceAdapter`的类：

- `CloseableProducerToDataSourceAdapter` 提供`create`方法创建实例，实现了`closeResult`方法，会在`AbstractDataSource`销毁时同时销毁收到的Result，**是主要使用的DataSouce**；
- `ProducerToDataSourceAdapter` 提供`create`方法创建实例，没有实现别的方法。**此DataSource仅仅用于[预加载图片](http://fresco-cn.org/docs/using-image-pipeline.html#)！**

###3.3 其他DataSource

####IncreasingQualityDataSource

它内部维持着一个`AbstractDataSource`（可以说是`CloseableProducerToDataSourceAdapter`）列表，**DataSource提供数据的清晰度由后往前递增**。

它会为列表中的每一个DataSource绑定一个DataSubscriber（`IncreasingQualityDataSourceSupplier.InternalDataSubscriber`），它负责保证每次只能获取清晰度更高的DataSource数据，获取数据同时会销毁数据清晰度更低的DataSource。

####FirstAvailableDataSource

它内部维持着一个`AbstractDataSource`（可以说是`CloseableProducerToDataSourceAdapter`）列表，它会返回这里面首先能获取到数据的DataSource。

同样，它也会为列表中的DataSource绑定DataSubscriber（`FirstAvailableDataSourceSupplier.InternalDataSubscriber`），如果数据加载成功，那么就设定指定DataSource为目标DataSource；如果加载数据失败，则跳转到列表下一个DataSource继续尝试加载。

####SettableDataSource

它继承AbstractDataSource，并将重写`settResult`、`setFailure`、`setProgress`在内部调用父类的相应函数，**但是修饰符变成了public**（原来是protected）。即使用`SettableDataSource`时可以在外部调用这三个函数设置DataSource状态。一般用于在获取DataSource失败时直接产生一个设置为Failure的DataSource。

关于数据源的其他内容，可以参考[数据源/订阅数据源](http://fresco-cn.org/docs/datasources-datasubscribers.html#_)。

[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md
[7]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(7)%20-%20%E8%A7%A3%E7%A0%81.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier
[Closeable]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer


