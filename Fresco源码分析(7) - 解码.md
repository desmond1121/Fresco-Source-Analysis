#Fresco源码分析(7) - 解码
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

本章是Fresco源码分析的最后一章，Fresco的解码是一大特色，不仅可以渐进式加载JPEG，支持EXIF、WEBP等格式图片，而且在android5.0以下会将解码后的Bitmap放在ashmem中提高效率。

##1 不同版本的实现

Fresco中针对不同版本使用了不同的图片解码工具，在分析它们之前，首先需要介绍几个[BitmapFactory.Options](http://developer.android.com/reference/android/graphics/BitmapFactory.Options.html)，它们对定制Bitmap的加载起到了很大的作用。

|名称|功能|
|:--:|:--:|
|inJustDecodeBounds|是否不加载图片而仅仅是获取图片参数|
|inDither|是否通过抖动技术增强画质低的图片效果|
|inPreferredConfig|Bitmap格式|
|inPurgeable|是否加载图片到ashmem中|
|inMutable|是否允许修改图片|
|inBitmap|是否重用Bitmap对象，在4.4之前只能重用尺寸相同的对象|

###1.1 ArtBitmapFactory

适用于Android 5.0及以上系统的解码工具。

它解码静态图片的流程可以描述为：



1. 在已分配的内存中查找

###1.2 DalvikBitmapFactory

适用于Android 2.3-5.0之间系统的解码工具。


###1.3 GingerbreadBitmapFactory

适用于Android 2.3系统的解码工具。

##1 解码实现

`ImageDecoder`封装了解码工具，它主要的被使用函数就是`decodeImage`，会根据图像格式进行相应解码。会依照JPEG, Gif, AnimatedWebp，静态图片的顺序检查图片格式，符合则调用相应函数（未知图片格式直接抛出异常）。

####1.1.n 自定义JPEG解码

这部分内容可以直接参考[渐进式JPEG图](http://fresco-cn.org/docs/progressive-jpegs.html#_)

##2 重温DecodeProducer

在[Producer流水线][6-2.4]一章的分析中，我仅仅介绍了起其功能。但是要了解解码的运行机制，我们需要重温一下它的代码。

###2.1 JobScheduler

Fresco中使用JobScheduler来管理解码任务（JobRunnable），它保证同一时间只有一个JobRunnable在执行，并且在一定时间内每个JobRunnable对象只会被执行一次。

通过构造函数可以直接初始化JobScheduler，它有以下两个重要函数，**均在this的同步块内执行**：

- `updateJob(EncodedImage encodedImage, boolean isLast)`更新需要被解码的EncodedImage；
- `scheduleJob()` 当JobRunnable没有被执行的时候，将它列入待执行队列，它会在能够执行的时候被执行；

###2.2 包装Consumer

在DecodeProducer中使用ProgressiveDecoder对Consumer进行初步包装，它会创造一个JobRunner来执行解码任务，并创建对应的JobScheduler。它会在下层Producer传数据上来的时候采取几项操作：

- 产生新结果时，调用JobScheduler的`scheduleJob()`函数进行解码，解码成功后调用上层Consumer的NewResult相应函数；
- 失败时，JobScheduler会释放内部保持的EncodedImage，并调用上层Consumer的失败响应函数；
- 取消时，JobScheduler会释放内部保持的EncodedImage，并调用上层Consumer的取消响应函数；

`ProgressiveDecoder`实现了最基础的解码函数`deCode(EncodedImage encodedImage, boolean isLast)`，它会尝试使用ImageDecoder来对EncodedImage进行解码

    private void doDecode(EncodedImage encodedImage, boolean isLast) {
        if (isFinished() || !EncodedImage.isValid(encodedImage)) {
            return;
        }

        try {
            long queueTime = mJobScheduler.getQueuedTime();
            int length = isLast ?
                    encodedImage.getSize() : getIntermediateImageEndOffset(encodedImage);
            QualityInfo quality = isLast ? ImmutableQualityInfo.FULL_QUALITY : getQualityInfo();

            mProducerListener.onProducerStart(mProducerContext.getId(), PRODUCER_NAME);
            CloseableImage image = null;
            try {
                image = mImageDecoder.decodeImage(encodedImage, length, quality, mImageDecodeOptions);
            } catch (Exception e) {
                Map<String, String> extraMap = getExtraMap(image, queueTime, quality, isLast);
                mProducerListener.
                        onProducerFinishWithFailure(mProducerContext.getId(), PRODUCER_NAME, e, extraMap);
                handleError(e);
                return;
            }
            Map<String, String> extraMap = getExtraMap(image, queueTime, quality, isLast);
            mProducerListener.
                    onProducerFinishWithSuccess(mProducerContext.getId(), PRODUCER_NAME, extraMap);
            handleResult(image, isLast);
        } finally {
            EncodedImage.closeSafely(encodedImage);
        }
    }


[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md
[6-2.4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md#24-功能producer
[7]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(7)%20-%20%E8%A7%A3%E7%A0%81.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier
[Closeable]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer