#Fresco源码分析(3) - DraweeView显示图层树
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

Fresco的源码中，DraweeView的介绍简洁明了：我就是把DraweeHierarchy显示到屏幕上的家伙。那我们来分析一下相关代码，看看它的逻辑是什么样的。

##1 时序图

首先可以用以下这个图初步理解SimpleDraweeView在调用了`setUri(Uri uri)`之后的流程：

![DraweeView](http://desmondyao.com/image/fresco/sequence_diagram_setUriSeq1.PNG)

可以将这张图描述为以下信息：

- DraweeView直接显示DraweeHierarchy；
- DraweeController根据Uri获取数据源`DataSource`，并绑定数据订阅者`DataSubscriber`；
- 当`DataSource`可以更新数据时通知`DataSubscriber`更新DraweeHierarchy。（在[Fresco源码分析(4) - 异步加载数据][4]中分析）

##2 基类DraweeView

`DraweeView<DH extends DraweeHierarchy>`是Fresco视图中的基类，使用的泛型必须是`DraweeHierarchy`，它继承了`ImageView`。

`DraweeView`内部的函数不多，**它们全部都是通过DraweeHolder的相应函数实现的。**主要有以下这几个：
- `void init(Context context)` 初始化`DraweeHolder`；
- `void setHierarchy(DH hierarchy)` 设置图层树，会调用 `super.setImageDrawable(mDraweeHolder.getTopLevelDrawable())`将图层树显示出来；
- `void setController(@Nullable DraweeController draweeController)` 设置`DraweeController`，像`setHierarchy`一样同样也会显示图层树；
- `Drawable getTopLevelDrawable()` 获取通过包装好的图层树(见[DraweeHierarchy构建图层][2])；
- `View`中的`onAttachedToWindow()`、 `onDetachedFromWindow()`、 `onStartTemporaryDetach()`、 `onFinishTemporaryDetach()` 四个回调函数，提供当视图被绑定/解绑到指定布局上时的回调函数，它们会触发`DraweeHolder`的`onDetach()`或`onAttach()`；
- `void onTouchEvent(MotionEvent event)` 提供触屏反应。

在使用它之前要注意官方的API注释中有这么一段话：

>*你只能在将DraweeView仅仅当做ImageView来使用时才调用`setImageXXX`函数。*

要记住它：无论什么形式的`DraweeView`，目标图片的设置最终都是通过`DraweeController`。我们会在下一章中介绍`DraweeController`，它是连接`DraweeHierarchy`与`Image Pipeline`的桥梁。虽然`setHierarchy`与`setController`都会显示出图层树，但是实际上**`setHierarchy`在显示图片的时候只是把`DraweeView`当做普通的`ImageView`使用**，要使用Fresco的缓存、加载机制，必须使用`DraweeController`。

不过所幸Fresco实现了`SimpleDraweeVew`帮我们来处理这些繁琐的过程，它封装了`Controller`的使用。

###2.1 DraweeHolder

`DraweeHolder`充斥在`DraweeView`的各个位置，每个`DraweeView`的函数都是由它的对应函数执行的。它随着`DraweeView`的产生而初始化。在深入了解视图绘制之前，我们有必要了解它是做什么的。

`DraweeHolder`是用来维持`DraweeHierarchy`和`DraweeController`之间的沟通的。通过`create( DH hierarchy, Context context)`来创建实例，`DraweeHierarchy`通过第一个参数赋值，其中第二个参数用于`registerWithContext(Context)`（该函数暂时没有完善好）。

它有几个主要使用的函数：
- `void setHierarchy(DH hierarchy)` 在`DraweeView.setHierarchy`中被调用，将DraweeHierarchy传给持有的DraweeController；
- `void setController(DraweeController draweeController)` 在`DraweeeView.setController`中被调用无条件解绑旧的Controller（如果存在的话），并将旧DraweeController的DraweeHierarchy设置为null，并调用`DraweeController.onDetach()`将它变为解除绑定状态；将持有的DraweeHierarchy（如果有的话）赋给新传入的DraweeController并调用`DraweeController.onAttach()`让它变为绑定状态。（**更换DraweeController确是一个非常耗时的过程，应该尽量避免为视图指定新的DraweeController，参考[官方文档](http://fresco-cn.org/docs/using-controllerbuilder.html#draweecontroller)）**
- `void attachController()` 若所属的`DraweeView`未绑定`DraweeController`，`DraweeController`成员变量不为空并且已经设置过图层树之后，调用该成员变量的`onAttach`方法；
- `void detachController()` 若所属的`DraweeView`已绑定`DraweeController`，`DraweeController`成员变量不为空，调用该成员变量的`onDetach`方法；
- `void attachOrDetachController()` 当已绑定`DraweeController`并且所属的`DraweeView`可见时，调用`attachController`；否则调用`detachController`。

特别地，`DraweeHolder`继承了`VisibilityCallback`，为`DraweeHierarchy.RootDrawable`提供回调：当图层的Visibility属性改变的时候对`DraweeController`调用`attachOrDetachController`操作，当图层不可见时释放资源。

###2.2 GenericDraweeView

`GenericDraweeView`就是使用`GenericDraweeHierarchy`图层树的视图，它承担了所有的xml属性交互工作。

*实际上在图层树上目前Fresco也只实现了一个`GenericDraweeHierarchy`，使用泛型是为了后续的开发便利。*

它会在初始化的时候调用`inflateHierarchy(Context context, AttributeSet attrs)`函数，从xml的中获取如下几个属性（如果存在的话）：
- `fadeDuation` 渐隐/渐显动画时间；
- `aspectRatio` 图片长宽比例，参考[官方文档](http://fresco-cn.org/docs/using-drawees-xml.html#wrap-content)；
- ``XXXImage`/`XXXImageScaleType` 各图层要显示的Drawable（除了目标显示图层）及它们的`ScaleType`；
- `RoundingParams`中的参数

在它的Measure过程中，会依次判断是否高度、宽度属性中有`wrap_content`，会将先判断到的属性更正为实际长度。**但是Fresco并不支持使用wrap_content。**如果你非要使用，只能在width/height中使用一侧，然后搭配`aspectRatio`使用。

**在GenericDraweeView获取完xml属性之后，它会通过`GenericDraweeHierarchyBuilder.build`创造一个与之对应的GenericDraweeHierarchy作为默认使用的图层树，并调用`setHierarchy`方法将其传递给`DraweeHolder`并显示出来。**

###2.3 SimpleDraweeView

在SimpleDraweeView中的函数就更少了，可以明确的说，它就只是将GenericDraweeHierarchy显示到UI界面上的空间而已。它比GenericDraweeView多出来的功能就是**它内部提供了最简单的DraweeController实现**。它有两个函数比较需要关注：

- `initialize`函数，这个函数会在`Fresco.initialize`中调用，目的就是初始化DraweeControllerBuilder，用于构建DraweeController。

- `setImageURI` 这个函数我们会经常调用，因此有必要看看它的实现和普通ImageView的`setImageXXX`方法有什么区别：

```java
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller = mSimpleDraweeControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
    setController(controller);
  }
```

而`setController`会调用`DraweeHolder.setController`，将图层树的控制权交给DraweeController，并显示出图层树。

##3 DraweeController

DraweeController是一个将Fresco中负责数据加载的组件组合起来并将信息反映到DraweeHierarchy的组件。它通过建造者模式初始化，基类`AbstractDraweeControllerBuilder`使用了四个泛型：（括号中为`PipelineDraweeControllerBuilder`所使用的类型）

- BUILDER 建造者类型(AbstractDraweeControllerBuilder)
- REQUEST 图像请求类型(ImageRequest)
- IMAGE 图像类型(CloseableReference<CloseableImage>)
- INFO 图像信息类型(ImageInfo)

在`.build()`中会初始化DraweeController。下面我们会先介绍几个关键概念，然后介绍DraweeController的初始化过程。

###3.1 ImageRequest

`ImageRequest`存储着Image Pipeline处理被请求图片所需要的有用信息(Uri、是否渐进式图片、是否返回缩略图、缩放、是否自动旋转等)。

**ImagePipeline仅仅用来装信息，而且一经初始化后就只能获取内容，无法改变内容（即Immutable）。** 初始化ImageRequest只能通过`ImageRequest.fromUri(Uri uri)`或`ImageRequestBuilder.build()`来实现。

我们来看看它内部存储着一些什么信息：

- `ImageType`：若为`ImageType.SMALL`，则所请求的图片会存储在专用小文件缓存中。具体参考[Fresco缓存](http://fresco-cn.org/docs/caching.html)；
- `SourceUri`：图片源Uri；
- `SourceFile`：图片文件地址（若是本地文件的话）；
- `ProgressiveRenderingEnabled` 若为true，则这个图片请求会返回质量递进的几次图片信息（渐进式图片）；
- `LocalThumbnailPreviewsEnabled` 若为true，则这个图片请求会在访问本地图片时先返回一个缩略图；
- `ResizeOptions` 缩放尺寸，仅支持JPEG，而且不是每次都需要在ImageRequest中设置缩放尺寸的，具体使用请参照[Fresco缩放和旋转图片](http://fresco-cn.org/docs/resizing-rotating.html#)；
- `AutoRotateEnabled` 是否允许图片旋转，具体参照[Fresco缩放和旋转图片](http://fresco-cn.org/docs/resizing-rotating.html#)；
- `RequestPriority` 这个请求的优先级：
- `LowestPermittedRequestLevel` 最低允许从哪层缓存中取数据，参考[最低请求级别](http://fresco-cn.org/docs/image-requests.html#)；
- `IsDiskCacheEnabled` 若为false，则此图片请求不会从文件缓存中获取数据；
- `PostProcessor` 在图片请求成功之后对图片的处理操作，参考[修改图片](http://fresco-cn.org/docs/modifying-image.html#_)。

DraweeController是使用ImageRequest来初始化数据订阅者的。`SimpleDraweeView`调用`setUri(Uri)`会产生一个默认的`ImageRequest`含有指定Uri信息，如果需要修改`ImageRequest`其他信息，必须手动创建`ImageRequest`，并在`PipelineDraweeControllerBuilder`调用`.build()`之前使用`.setImageRequest`设置它。

###3.2 可关闭的引用

Facebook在Java中实现了具有引用计数功能的类：`SharedReference<T>`（注意不是Android里的SharedPreference）。它将类型为`T`的对象进行包装，为其实现增加引用数、减少引用数、删除引用的功能。

特别地，它会使用专门用于实现释放资源的接口：`ResourceReleaser<T>`，它内部只有一个函数：`void release(T object)`。

当然，但是直接让程序员直接去操作它很可能会出现问题。所以`CloseableReference`出现了，它为任何实现了`Closeable`类的对象封装了引用计数、回收引用的功能。

`CloseableReference`中有几个主要函数：

- `CloseableReference<T> of(T object, ResourceReleaser<T> releaser)` **用于初始化引用计数（而不是使用构造函数）**，该函数会新建一个`SharedReference`将对象包装起来，引用计数为1。若不传releaser，会使用默认的`ResourceReleaser`，调用`Closeable.close()`函数回收`Closeable`引用；
- `CloseableReference<T> clone()` 用于添加对象引用，该函数会新建一个`CloseableReference`，同时持有的`SharedReference`引用+1， **不会创建`SharedReference`对象**；
- `T get()` 返回引用对象；
- `void close()` 减少一个引用计数，若引用计数减为0，则调用releaser的`release(T object)`将对象释放。**一旦创建了一个CloseableReference，当持有者离开作用域时就必须调用这个函数！（finally函数块是释放资源工作最好的地方）**

最好不要直接使用这个工具，如果非用不可的话，你需要谨慎地操作它。使用方法参考[Fresco中文文档](http://fresco-cn.org/docs/closeable-references.html#_)。

Fresco中定义了`CloseableImage`，它会在`finalize`的时候调用`close()`，有这两个类继承了它：

- `CloseableStaticBitmap` 它内部持有一个`CloseableReference<Bitmap>`包装目标`Bitamp`，以及关于图像质量、旋转角度的信息。在`close()`调用的时候会调用`CloseableReference`的`close()`函数释放资源，**释放的原理是Bitmap.recycle()**；
- `CloseableAnimatedBitmap` 它内部持有一个`List<CloseableReference<Bitmap>>`包装起每一帧的`Bitmap`，还存有每一帧的时长。在`close()`调用的时候会递归释放列表资源。

###3.3 数据订阅

Fresco中使用DataSource与DataScriber进行异步数据请求。DataSubscriber具有以下几个函数：

- `onNewResult` 在接收到新的数据；
- `onFailure` 数据加载失败；
- `onCancellation` 数据请求被取消；
- `onProgressUpdate` 加载进度更新。

DataSource在接受到Image Pipeline提供的数据时调用`notifyDataSubscribers`使DataSubscriber做出反应。

更多对于数据订阅者的分析见[Fresco源码分析(4) - 异步加载数据][4]。

###3.4 初始化Draweetroller

在`AbstractDraweeControllerBuilder`（`DraweeControllerBuilder`的基类）的`build()`函数中会调用继承类实现的`obtainController()`函数，在默认使用的`PipelineDraweeControllerBuilder`中，它会做三件重要的事情：

- `obtainDataSourceSupplier()` 根据Uri获取数据源`DataSource`的[Supplier][Supplier]；
- `generateUniqueControllerId()` 获取独一无二的Controller标识（使用静态递增的`AtomicLong`变量记录唯一标识）；
- `getCallerContext()` 获取调用者的`Context`。

之后将上面得到的三个变量赋值给Controller，若是第一次设置`DraweeController`，还会相应地初始化这几个组件（若存在的话）：

- `RetryManager` 管理失败重试的组件；
- `GestureDetector` 传递触屏事件的组件；
- `DeferredReleaser` 向主线程消息队列中添加释放资源任务的组件；
- `ControllerListener` 设置回调函数，具体参考 **3.5 使用ControllerListener**；
- `DraweeHierarchy` 设置图层树。

在`setController`调用后，如果传入的`DraweeController`发生了`onAttach`，它就会调用`subtitRequest()`提交数据请求（如果还没有提交的话），我们来看看这个函数是怎么实现的：

    protected void submitRequest() {

        //一些初始化工作

        final String id = mId;
        final boolean wasImmediate = mDataSource.hasResult();
        final DataSubscriber<T> dataSubscriber =
            new BaseDataSubscriber<T>() {
              @Override
              public void onNewResultImpl(DataSource<T> dataSource) {
                boolean isFinished = dataSource.isFinished();
                float progress = dataSource.getProgress();
                T image = dataSource.getResult();
                if (image != null) {
                  onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
                } else if (isFinished) {
                  onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
                }
              }
              @Override
              public void onFailureImpl(DataSource<T> dataSource) {
                onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
              }
              @Override
              public void onProgressUpdate(DataSource<T> dataSource) {
                boolean isFinished = dataSource.isFinished();
                float progress = dataSource.getProgress();
                onProgressUpdateInternal(id, dataSource, progress, isFinished);
              }
            };
        mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
    }

从这段代码中我们就可以看出初步的逻辑了，它会创建一个`DataScriber`并将其绑定到`DataSource`上，随后只要`DataSource`处理好图片就会为绑定的`DataScriber`发布消息。通过在`AbstractDraweeController`中定义的`onFailureInternal`、`onNewResultInternal`、`onProgressUpdateInternal`来对DraweeHierarchy做相应的改动，从而控制显示层。

###3.5 使用ControllerListener

`ControllerListener` 提供`DraweeController`主要事件的回调功能，主要有这几个事件：

 + `onSubmit` 在提交图片请求时调用；
 + `onFinalImageSet` 最终图片加载完成时调用；
 + `onIntermediateImageSet` 任何渐进式图片的过渡图片加载完成时调用；
 + `onIntermediateImageFailed` 过渡图片加载失败时调用；
 + `onFailure` 最终图片加载失败时调用；
 + `onRelease` 释放图片资源时调用。

你可以继承它并在`DraweeControllerBuilder`中设置，从而实现一些自定义的提醒事件。

##4 类图

![DraweeView Diagram](http://desmondyao.com/image/fressco/class_diagram_draweeview.PNG)

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
