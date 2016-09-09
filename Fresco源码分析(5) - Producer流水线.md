#Fresco源码分析(5) - Producer流水线
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

Producer是缓存与DataSource之间的桥梁，它非常重要！在这一章的内容中，我们来分析Fresco中的Producer是怎么通过ImageRequest建立起数据请求通道的。你可以在[Wiki-Producer][Producer]中对Producer&Consumer模式及它们在Fresco中的原型有个初步的理解。

##1 Fresco缓存机制

在阅读后面的内容前，你需要先对Fresco的缓存有一个感性的理解。这里官方文档[Image Pipeline介绍](http://fresco-cn.org/docs/intro-image-pipeline.html#_)及[缓存](http://fresco-cn.org/docs/caching.html#_)已经说得很好了，读者可以直接参考它。

Fresco会一级一级地去检查缓存，一共有三级缓存：

1. 已解码内存缓存；
2. 未解码内存缓存；
3. 文件缓存。

如果这三个缓存都没有命中，则会从网络或者本地加载，加载完成再缓存到各个缓存中。

###1.1 缓存载体

**未解码内存缓存的载体是`EncodedImage`**，它封装了未解码图片的所有字节码，包括图片数据、尺寸、旋转角度和缩放尺寸。它使用`PooledByteBuffer`存储字节码，可以直接通过`CloseableReference<PooledByteBuffer>`构造。

它有这两个主要函数：

- `getInputStream()` 返回存储着字节码的InputStream；
- `parseMetaData()` 解析字节码中的尺寸、旋转角度和缩放尺寸到成员变量中。

**已解码内存缓存的载体是`CloseableBitmap`**（关于CloseableBitmap见[可关闭的引用][3-3.2]）。

更多缓存机制可以参考后续分析[Fresco源码分析(6)-缓存][6]，此处你仅仅需要理解缓存结构即可。

##2 元Producer

**Fresco将Producer组织成流水线来进行多层内容顺序访问。** 我做了一个流程图，供读者参考。略有精简，不过已经能够代表大概意思：

![ProducerSequence](http://desmondyao.com/image/fresco/producer_cache_sequence.PNG)

图中每一个方框都代表一个Producer，**蓝色框内的Producer会在缓存中取数据。**在产生一个Uri的时候，最先在Bitmap内存缓存中查找，若命中则返回，否则将会调用下层Producer的`produceResult`函数。大部分Producer都是其中间作用，有几个在末端的Producer是在之前所有缓存都没有命中后要去文件或网络上获取数据的，我将他们称做**元Producer**。一共有两大类，他们是LocalProducer（负责本地数据存取）及NetworkProducer（负责网络数据存取）。

###2.1 本地文件获取

首先介绍从本地获取文件的Producer的基类-`LocalFetchProducer`，我们来看看它的`produceResults`中做了什么事：

    final StatefulProducerRunnable cancellableProducerRunnable =
        new StatefulProducerRunnable<EncodedImage>(
            consumer,
            listener,
            getProducerName(),
            requestId) {

          @Override
          protected EncodedImage getResult() throws Exception {
            EncodedImage encodedImage = getEncodedImage(imageRequest);
            if (encodedImage == null) {
              return null;
            }
            encodedImage.parseMetaData();
            return encodedImage;
          }

          @Override
          protected void disposeResult(EncodedImage result) {
            EncodedImage.closeSafely(result);
          }
        };

    //...

    mExecutor.execute(cancellableProducerRunnable);

里面有两个关键点：

1. 取数据的工作由`StatefulProducerRunnable`完成，它继承了`Runnable`。
2. 在`mExecutor`上执行获取数据的任务，在`DefaultExecutorSupplier`中可以知道**用于本地读取文件的`Executor`是容量为2的线程池**。

它没有实现`getEncodedImage`。这也是合理的，因为获取不同种类资源的方法不同。继承LocalFetchProducer的本地Producer可以通过重写`getEncodedImage`实现获取各类资源。

####2.1.1 StatefulRunnable

`StatefulProducerRunnable`继承了`StatefulRunnable`。首先看StatefulRunnable中的run函数：

    @Override
    public final void run() {
        if (!mState.compareAndSet(STATE_CREATED, STATE_STARTED)) {
          return;
        }
        T result;
        try {
          result = getResult();
        } catch (Exception e) {
          mState.set(STATE_FAILED);
          onFailure(e);
          return;
        }

        mState.set(STATE_FINISHED);
        try {
          onSuccess(result);
        } finally {
          disposeResult(result);
        }
    }

它会在运行的时候调用`getResult`，失败则会调用`onFailure`，成功会调用`onSuccess`，最后将获取的数据释放。

那我们看看`StatefulProducerRunnable`中的相应函数：

    protected void onSuccess(T result) {
        //..监听事件
        mConsumer.onNewResult(result, true);
    }

    protected void onFailure(Exception e) {
        //..监听事件
        mConsumer.onFailure(e);
    }

    protected void onCancellation() {
        //..监听事件
        mConsumer.onCancellation();
    }

而留下`getResult`与`disposeResult`未实现。这下就很明确了，它在数据获取成功、失败、取消时分别会通知Comsumer执行相应函数。至于要怎么获取、释放数据，交由具体实现者来定。

###2.2 几种基本的LocalProducer

LocalProducer提供了将`InputStream`转化成`EncodedImage`的函数`getByteBufferBackedEncodedImage`供继承类使用，我们来看看几个继承`LocalProducer`的类是怎么重写`getEncodedImage`的吧：

- `LocalAssetFetchProducer` 通过AssetManager获取ImageRequest对象的输入流及对象字节码长度，将它转换为`EncodedImage`；
- `LocalContentUriFetchProducer` 若Uri指向联系人，则获取联系人头像；若指向相册图片，则会根据是否传入ResizeOption进行**一定缩放**（这里不是完全按ResizeOption缩放）；若止这两个条件都不满足，则直接调用`ContentResolver`的函数`openInputStream(Uri uri)`获取输入流并转化为`EncodedImage`；
- `LocalFileFetchProducer` 直接通过指定文件获取输入流，从而转化为`EncodedImage`；
- `LocalResourceFetchProducer` 通过`Resources`的函数`openRawResources`获取输入流，从而转化为`EncodedImage`。

下面两个类不继承`LocalProducer`，他们获取数据也是在后台线程中执行，但是有比较特殊的功能，在此一起介绍：

- `LocalExifThumbnailProducer` 可以获取Exif图像的Producer；
- `LocalVideoThumbnailProducer` 可以获取视频缩略图的Producer。

###2.3 网络文件获取

####2.3.1 NetworkFetcher

`NetworkFetcher`是Fresco用以获取数据，计算下载进度，提供下载回调函数的工具。它使用`fetch(FetchState fetchState, Callback callback)`来进行网络请求，其中两个参数的意义分别为：

- `FetchState` 提供了Consumer及ProducerContext，用于数据回调对象及获取数据请求参数；
- `Callback` 这个借口定义在了`NetworkFetcher`中，提供了下载成功、失败、取消的回调函数接口。

由于Fresco中默认使用HttpUrlConnection进行网络请求，我们直接分析HttpUrlConnectionNetworkFetcher，首先看它的`fetch`函数：

    public void fetch(final FetchState fetchState, final Callback callback) {
        final Future<?> future = mExecutorService.submit(
            new Runnable() {
              @Override
                  public void run() {
                    //初始化工作

                    while (true) {
                      String nextUriString;
                      String nextScheme;
                      InputStream is;
                      try {
                        URL url = new URL(uriString);
                        connection = (HttpURLConnection) url.openConnection();
                        nextUriString = connection.getHeaderField("Location");
                        nextScheme = (nextUriString == null) ? null : Uri.parse(nextUriString).getScheme();
                        if (nextUriString == null || nextScheme.equals(scheme)) {
                          is = connection.getInputStream();
                          callback.onResponse(is, -1);
                          break;
                        }
                        uriString = nextUriString;
                        scheme = nextScheme;
                      } catch (Exception e) {
                        callback.onFailure(e);
                        break;
                      } finally {
                        if (connection != null) {
                          connection.disconnect();
                        }
                      }
                  }
              }
            });

我们可以看到它使用[Future](http://developer.android.com/reference/java/util/concurrent/Future.html)来进行异步请求。当Http返回的头中有重定向（返回字节Location）时则使用新url，否则就获取该url进行读取数据。调用`callback.onResponse`处理数据，当失败时调用`callback.onFailure`处理失败，最后将连接关闭。**使用HttpUrlConnectionNetworkFetcher返回的数据总长度为-1，即后续计算下载进度时是使用近似值，下载225kb后即达到100%。具体参考`NetworkFetchProducer.calculateProgress`的注释。**

并且在后面加了一段代码，当请求取消的时候让Future取消执行，并调用`callback.onCancellation`：

        fetchState.getContext().addCallbacks(
            new BaseProducerContextCallbacks() {
              @Override
              public void onCancellationRequested() {
                if (future.cancel(false)) {
                  callback.onCancellation();
                }
              }
            });

注：**默认的网络请求线程池容量为3，无法通过API修改**。

###2.3.2 NetworkFetchProducer

`NetworkFetchProducer`是专门用于在网络上获取数据的Producer。我们看看它的`produceResults`代码：

    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(
        fetchState, new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
            NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
          }

          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
        });

它一开始将Consumer与ProducerContext封装进FetchState中，并定义Callback让NetworkFetcher在产生结果、失败、取消请求时分别调用自身的对应函数。

当Failure、Cancellation时下几乎就是简单的调用Consumer相应函数而已，主要就看onResponse函数：

    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();

我们看到当InputStream中还有数据时就会想outputStream中写，计算并更新进度。在`maybeHandleIntermediateResult`会判断当两次获取数据间隔超过100ms即会通知Consumer更新一次数据。最后如果InputStream读取完了会再通知Consumer读取一次数据。

###2.3 缓存Producer

这类Producer负责从缓存中寻找数据，在初始化都会传入一个nextProducer，当没有获取到缓存时调用下一个Producer的`productResult(Consumer consumer, ProducerContext context)`方法。主要有这几种：

- `BitmapMemoryCacheGetProducer` 它是一个Immutable的Producer，仅用于包装后续Producer；
- `BitmapMemoryCacheProducer` 在已解码的内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存；
- `BitmapMemoryCacheKeyMultiplexProducer` 是`MultiplexProducer`的子类，nextProducer为`BitmapMemoryCacheProducer`，将多个拥有相同已解码内存缓存键（具体见[Fresco源码分析(6) - 缓存][6]）的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据；
- `PostprocessedBitmapMemoryCacheProducer` 在已解码的内存缓存中寻找PostProcessor处理过的图片。它的nextProducer都是`PostProcessorProducer`，因为如果没有获取到被PostProcess的缓存，就需要对获取的图片进行PostProcess。；若未找到，则在nextProducer中获取数据；
- `EncodedMemoryCacheProducer` 在未解码的内存缓存中寻找数据，如果找到则返回，使用结束后释放资源；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存；
- `EncodedCacheKeyMultiplexProducer` 是`MultiplexProducer`的子类，nextProducer为`EncodedMemoryCacheProducer`，将多个拥有相同未解码内存缓存键（具体见[Fresco源码分析(6) - 缓存][6]）的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据；
- `DiskCacheProducer` 在文件内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存

###2.4 功能Producer

这类Producer的初始化也都会传入一个nextProducer，它们的作用只是对nextProducer产生的结果进行处理，主要有这几种：

- `MultiplexProducer` 将多个拥有相同CacheKey（具体见[Fresco源码分析(6) - 缓存][6]）的ImageRequest进行“合并”，让他们从都从nextProducer中获取数据；
- `ThreadHandoffProducer` 将nextProducer的`produceResult`方法放在后台线程中执行（**线程池容量为1**）；
- `SwallowResultProducer` 将nextProducer的获取的数据“吞”掉，回在Consumer的onNewResult中传入null值；
- `ResizeAndRotateProducer` 将nextProducer产生的EncodedImage根据EXIF的旋转、缩放属性进行变换（**如果对象不是JPEG格式图像，则不会发生变换**）；
- `PostProcessorProducer` 将nextProducer产生的EncodedImage根据PostProcessor进行修改，关于PostProcessor详见[修改图片](http://fresco-cn.org/docs/modifying-image.html#_)；
- `DecodeProducer` 将nextProducer产生的EncodedImage解码。**解码在后台线程中执行，可以在ImagePipelineConfig中通过`setExecutorSupplier`来设置线程池数量，默认为最大可用的处理器数**；
- `WebpTranscodeProducer` 若nextProducer产生的EncodedImage为WebP格式，则将其解码成`DecodeProducer`能够处理的EncodedImage。解码在后代进程中进行。

以上所有的Producer（包括元Producer）都是从`ProducerFactory`中新建的，有兴趣的读者可以自行再去探索。

###2.5 包装Consumer

**Producer使用数据获取时向下传递，Consumer得到结果时是向上传递。**所以几乎每个Producer都会将上层Producer传下来的Consumer进行包装，在他们的NewResult中增加一些功能，由此来达到自己的目的。我们简单看个例子，如果BitmapMemoryCacheProducer在已解码的内存缓存中没有找到数据，它就会调用nextProducer的`productResult(Consumer consumer, ProducerContext context)`办法。但是它会对传入的Consumer进行一定包装，我们看看相应代码：

    protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(
            final Consumer<CloseableReference<CloseableImage>> consumer,
            final CacheKey cacheKey) {
        return new DelegatingConsumer<
                CloseableReference<CloseableImage>,
                CloseableReference<CloseableImage>>(consumer) {
            @Override
            public void onNewResultImpl(CloseableReference<CloseableImage> newResult, boolean isLast) {
                if (newResult == null) {
                    if (isLast) {
                        getConsumer().onNewResult(null, true);
                    }
                    return;
                }

                if (newResult.get().isStateful()) {
                    getConsumer().onNewResult(newResult, isLast);
                    return;
                }

                if (!isLast) {
                    CloseableReference<CloseableImage> currentCachedResult = mMemoryCache.get(cacheKey);
                    if (currentCachedResult != null) {
                        try {
                            QualityInfo newInfo = newResult.get().getQualityInfo();
                            QualityInfo cachedInfo = currentCachedResult.get().getQualityInfo();
                            if (cachedInfo.isOfFullQuality() || cachedInfo.getQuality() >= newInfo.getQuality()) {
                                getConsumer().onNewResult(currentCachedResult, false);
                                return;
                            }
                        } finally {
                            CloseableReference.closeSafely(currentCachedResult);
                        }
                    }
                }

                CloseableReference<CloseableImage> newCachedResult =
                        mMemoryCache.cache(cacheKey, newResult);
                try {
                    if (isLast) {
                        getConsumer().onProgressUpdate(1f);
                    }
                    getConsumer().onNewResult(
                            (newCachedResult != null) ? newCachedResult : newResult, isLast);
                } finally {
                    CloseableReference.closeSafely(newCachedResult);
                }
            }
        };
    }

我们看到有它将Consumer包装起来之后，在下层Consumer传递上来数据时按以下顺序处理：

1. 数据为空，直接传给上层Consumer处理并返回；
2. 数据存储着某种状态（如动态图存储着当前浏览帧）时，不进行缓存，直接传给上层Consumer处理并返回；
3. 当没有结束传递时，如果收到数据的质量大于缓存中对应数据的质量（如果存在的话）时，则传给上层Consumer处理并返回；
4. 数据传递结束，将得到的数据缓存起来，更新进度，通知上层Consumer处理。

##3 Producer流水线

`ProducerSequenceFactory`是专门将生成各类链接起来的Producer，根据其中的逻辑，我将可能涉及层次最深的Uri——网络Uri的Producer链在此列出，它会到每个缓存中查找数据，最后如果都没有命中，则会去网络上下载。

|顺序|Producer|是否必须|功能|
|:--:|:--:|:--:|:--|
|1|**PostprocessedBitmapMemoryCacheProducer**|否|在Bitmap缓存中查找被PostProcess过的数据|
|2|PostprocessorProducer|否|对下层Producer传上来的数据进行PostProcess|
|3|BitmapMemoryCacheGetProducer|是|使Producer序列只读|
|4|ThreadHandoffProducer|是|使下层Producer工作在后台进程中执行|
|5|BitmapMemoryCacheKeyMultiplexProducer|是|使多个相同已解码内存缓存键的ImageRequest都从相同Producer中获取数据|
|6|**BitmapMemoryCacheProducer**|是|从已解码的内存缓存中获取数据|
|7|DecodeProducer|是|将下层Producer产生的数据解码|
|8|ResizeAndRotateProducer|否|将下层Producer产生的数据变换|
|9|EncodedCacheKeyMultiplexProducer|是|使多个相同未解码内存缓存键的ImageRequest都从相同Producer中获取数据|
|10|**EncodedMemoryCacheProducer**|是|从未解码的内存缓存中获取数据|
|11|**DiskCacheProducer**|是|从文件缓存中获取数据|
|12|WebpTranscodeProducer|否|将下层Producer产生的Webp（如果是的话）进行解码|
|13|NetworkFetchProducer|是|从网络上获取数据|

我们看到所有的获取数据操作都通过ThreadHandoffProducer包装到了后台进程中执行。加粗的Producer是会在缓存中查找数据的Producer，每一级缓存如果命中，则会直接返回结果而不会再传递到下一个Producer中。

##4 结语

Producer是理解Fresco处理缓存的一个很好入手点，本章中仅仅是分析其脉络及一些基础原理，如果有兴趣的读者可以自行再去探究。

[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md
[7]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(7)%20-%20%E8%A7%A3%E7%A0%81.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier
[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
