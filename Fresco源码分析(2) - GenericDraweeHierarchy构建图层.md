#Fresco源码分析(2) - GenericDraweeHierarchy构建图层

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121) 

在本篇的内容中，作者将初步介绍Fresco是怎么构建图像层次的。

##1 引言

`DraweeHierarchy`是所有`Hierarchy`的父接口，它内部只提供了一个基本而又不可缺失的功能：获取图层树的父节点图层。不过仅仅只有这个功能是不够的，Fresco紧接着用接口`SettableHierarchy`来继承它，声明一些具体的功能：

- `void reset()` 重置所有图层（慎用）
- `void setImage(Drawable drawable, float progress, boolean immediate)` 设置目标显示图片，`progress`表示图片的加载质量（在渐进式图中使用），`immediate`设置为true时图片会马上显示（而不是有一个渐变的过程）。**但是如果你使用这个函数来设置目标显示图片，将意味着你在加载这张图片的时候放弃了Fresco的Image Pipeline加载图片的方式。**
- `void setProgress(float progress, boolean immediate)`  设置`ProgressBar`的进度。当`progress`为1时会隐藏`ProgressBar`，同样`immediate`设置为true时进度条会马上消失。
- `void setFailure(Throwable throwable)` 图片加载失败的回调，可以用来显示失败提示图片。参数为加载失败时抛出的异常。
- `void setRetry(Throwable throwable)` 图片加载失败但是还希望再次尝试加载时的回调，可以用来显示重试提示图片。参数为加载失败时抛出的异常。
- `void setControllerOverlay(Drawable drawable)` 由于特殊原因要用指定图层覆盖已有图层时的回调，传入的参数会显示在`ArrayDrawable`的最上方。

这几个函数都会在后面起到比较大的作用。
在接下来的内容中会介绍本节的主角：**GenericDraweeHierarchy**。它实现了`SettableHierarchy`接口，你可以从这个类中看到大部分Fresco处理图层的逻辑。

##2 图层封装者 - GenericDraweeHierarchy

请记住这句话：**GenericDraweeHierarchy**只是负责装载每个图层信息的载体。**如果你直接使用它去显示图片，那就意味着你将放弃Fresco提供的加载与缓存机制。**你可以认为这么做之后`SimpleDraweeView`就退化成了一个简单的`ImageView`，只会将`ArrayDrawable`中的所有设置的图片按顺序显示出来。具体的细节我们将在[Fresco源码分析(3) - DraweeView显示图层树][3]中讨论。

首先看几个成员变量：

```java
  /* 占位图层 */
  private final int mPlaceholderImageIndex;
  /* 进度条图层 */
  private final int mProgressBarImageIndex;
  /* 目标显示图层 */
  private final int mActualImageIndex;
  /* 重试图层 */
  private final int mRetryImageIndex;
  /* 失败图层 */
  private final int mFailureImageIndex;
  /* 控制覆盖图层 */
  private final int mControllerOverlayIndex;
```

简洁明了有木有！没错，这个`GenericDraweeHierarchy`就是封装与维护Drawable层次的家伙！你需要牢记以上这六种图层名字，它是Fresco的视图显示中最主要的六个图层。

###2.1 建造者模式

如果你经常使用Fresco，你就会发现它的设计之中充斥着建造者模式。由于Fresco中的对象初始化经常是比较复杂的，建造者模式能为开发者在创建实例上省去很多功夫。

`1GenericDraweeHierarchy`的建造者是`GenericDraweeHierarchyBuilder`。它内部维持着许多图层属性，主要有这两种：
- 默认的变量（渐变动画时间、`ScaleType`）
- 程序的Resources实例
- 圆角矩形容器的一些参数
- 要放到每个图层容器中的Drawable实例及图层要应用的`ScaleType`、`Matrix`等等。

这个Builder内部有大量的getter与setter，你可以为每个图层指定Drawable、ScaleType，以及目标显示图层还可以设置Matrix、Focus（配合`ScaleType`为`FOCUS_CROP`时使用）、ColorFilter。

###2.2 初始化图层

我们来看一下`GenericDraweeHierarchy`，从中能够理解Fresco是怎么初始化图层的。

```java
  GenericDraweeHierarchy(GenericDraweeHierarchyBuilder builder) {
    mResources = builder.getResources();
    // 获取圆角参数
    mRoundingParams = builder.getRoundingParams();.
    // 初始化图层数为0
    int numLayers = 0;

    int numBackgrounds = (builder.getBackgrounds() != null) ? builder.getBackgrounds().size() : 0;
    int backgroundsIndex = numLayers;
    numLayers += numBackgrounds;
```
在这段代码中我们可以看到最开始初始化的是背景图层（顶层图层），会根据是否传入背景图层来判断图层数是否增减。再接着往下看：
```
    Drawable placeholderImageBranch = builder.getPlaceholderImage();
    if (placeholderImageBranch == null) {
      placeholderImageBranch = getEmptyPlaceholderDrawable();
    }
    placeholderImageBranch = maybeApplyRoundingBitmapOnly(
        mRoundingParams,
        mResources,
        placeholderImageBranch);
    placeholderImageBranch = maybeWrapWithScaleType(
        placeholderImageBranch,
        builder.getPlaceholderImageScaleType());
    mPlaceholderImageIndex = numLayers++;
```

在这段代码中，它对占位图层进行了以下处理：
- 获取图层Drawable资源，如果没有设置，它将创建一个透明图层。
- 根据圆角参数对图片进行圆角处理。
- 将待显示的Drawable资源包装进一个`ScaleTypeDrawable`中，处理缩放逻辑（关于`ScaleTypeDrawable`可以参考[Fresco源码分析(1) - 图像层次与各类Drawable][1]）。
- 记录图层在`ArrayDrawable`中的index，图层数量加一。

我们再看看目标显示图层的处理逻辑，与占位图层的处理有什么区别：
```
    Drawable actualImageBranch = null;
    mActualImageSettableDrawable = new SettableDrawable(mEmptyActualImageDrawable);
    actualImageBranch = mActualImageSettableDrawable;
    actualImageBranch = maybeWrapWithScaleType(
        actualImageBranch,
        builder.getActualImageScaleType(),
        builder.getActualImageFocusPoint());
    actualImageBranch = maybeWrapWithMatrix(
        actualImageBranch,
        builder.getActualImageMatrix());
    actualImageBranch.setColorFilter(builder.getActualImageColorFilter());
    mActualImageIndex = numLayers++;
```

与占位图层有区别的是它在显示图上多加了一了`SettableDrawable`容器（正常图层只有一个`ScaleTypeDrawable`容器），没有进行圆角处理。由于可以后续改变图像内容，它**直接使用了默认的透明图来初始化图层**，而且它还拥有ColorFilter、Matrix等特权。

需要注意的一点：在不显式指定图层内容的时候，**占位图层、目标显示图层、控制覆盖图层将会创建透明图层实例，其他图层不会创建实例。**  并且只有在指定内容的时候图层数量才会增加，除此以外其他图层与占位图、目标图层的初始化没有什么区别。

在初始化完基本图层之后，那我们接着看余下初始化过程：
```
    // overlays
    int overlaysIndex = numLayers;
    int numOverlays =
        ((builder.getOverlays() != null) ? builder.getOverlays().size() : 0) +
            ((builder.getPressedStateOverlay() != null) ? 1 : 0);
    numLayers += numOverlays;

    // controller overlay
    mControllerOverlayIndex = numLayers++;
```

这部分是初始化覆盖图层及控制覆盖图层。如果没有设置覆盖图层，他们不会被初始化。不过控制覆盖图层是会被初始化成透明图层的。

```
    // array of layers
    Drawable[] layers = new Drawable[numLayers];
    if (numBackgrounds > 0) {
      int index = 0;
      for (Drawable background : builder.getBackgrounds()) {
        layers[backgroundsIndex + index++] =
            maybeApplyRoundingBitmapOnly(mRoundingParams, mResources, background);
      }
    }
    if (mPlaceholderImageIndex >= 0) {
      layers[mPlaceholderImageIndex] = placeholderImageBranch;
    }
    
    // 各图层赋值
    
    if (mFailureImageIndex >= 0) {
      layers[mFailureImageIndex] = failureImageBranch;
    }
    if (numOverlays > 0) {
      int index = 0;
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[overlaysIndex + index++] = overlay;
        }
      }
      //按下时加在图片上的的覆盖图层
      if (builder.getPressedStateOverlay() != null) {
        layers[overlaysIndex + index++] = builder.getPressedStateOverlay();
      }
    }
    if (mControllerOverlayIndex >= 0) {
      layers[mControllerOverlayIndex] = mEmptyControllerOverlayDrawable;
    }
```
这段代码的意思也是简明易懂，它新建个Drawable数组，将存在的图层依次向里面添加。如果你了解了上一章的内容，你自然知道接下来做的是什么：用这个数组来初始化`FadeDrawable`，初始化整个视图层次。接着对图层做圆角处理（需要的话）。

```
    // 初始化FadeDrawable
    mFadeDrawable = new FadeDrawable(layers);
    mFadeDrawable.setTransitionDuration(builder.getFadeDuration());

    // 圆角处理
    Drawable maybeRoundedDrawable =
        maybeWrapWithRoundedOverlayColor(mRoundingParams, mFadeDrawable);

    // 包装成RootDrawable实例
    mTopLevelDrawable = new RootDrawable(maybeRoundedDrawable);
    mTopLevelDrawable.mutate();

    resetFade();
  }

```
在`resetFade()`中将占位图 、背景图层、覆盖图层显示出来。

###2.3 需要注意的一点：一个Drawable实例只能与一个DraweeHierarchy绑定！

如果绑定了多个DraweeHierarchy，会出问题。由于在初始化过程中它将Drawable数组赋值给`FadeDrawable`，而`FadeDrawable`又继承于`ArrayDrawable`，它会在初始化的时候为每个数组Drawable的`Drawable.Callback`设置为自己，若是`TransfromAwareDrawable`的话还会设置自己为`TransformCallback`。而我们可以在它的`setDrawable(int index, Drawable drawable)`函数中看到这么一段代码：

```java
    if (drawable != mLayers[index]) {
      if (mIsMutated) {
        drawable = drawable.mutate();
      }
      DrawableUtils.setCallbacks(mLayers[index], null, null);
      DrawableUtils.setCallbacks(drawable, null, null);
      DrawableUtils.setDrawableProperties(drawable, mDrawableProperties);
      DrawableUtils.copyProperties(drawable, mLayers[index]);
      DrawableUtils.setCallbacks(drawable, this, this);
      mIsStatefulCalculated = false;
      mLayers[index] = drawable;
      invalidateSelf();
    }
```
可以看出，在替换图片时，它会将原来Drawable的这些回调都设置为null。由此很可能会导致Bug，请参考我的一篇翻译文章：[Android LayerDrawable 和 Drawable.Callback](https://github.com/bboyfeiyu/android-tech-frontier/blob/master/issue-24/Android%20LayerDrawable%20%E5%92%8C%20Drawable.Callback.md)。

##3 类图

由于类中方法、变量过多，作者对其做了大量精简，仅用于参考设计层次。

![Class Diagram](http://desmondyao.com/image/fresco/class_diagram_hierarchy.PNG)

[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98.md
[7]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(7)%20-%20%E8%A7%A3%E7%A0%81.md
