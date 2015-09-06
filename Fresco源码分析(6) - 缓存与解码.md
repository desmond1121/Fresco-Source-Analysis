#Fresco源码分析(6) - 缓存与解码
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

本章是Fresco源码分析的最后一章，缓存与解码理解Fresco内存操作的最好入手点。

##1 缓存

Fresco中一共有三级缓存，其中前两级内存缓存都存储在Java堆上，本地缓存存储在本地文件目录中。

###1.1 CacheKey

Fresco中专门用于缓存键的接口，有这两种类实现了`CacheKey`：

- `BitmapMemoryCacheKey` 用于已解码的内存缓存键，会对Uri字符串、缩放尺寸、解码参数、PostProcessor等关键参数进行hashCode作为唯一标识；
- `SimpleCacheKey` 普通的缓存键实现，使用传入字符串的hashCode作为唯一标识，所以需要保证相同键传入字符串相同。

###1.2 内存缓存

已解码的内存缓存（BitmapMemoryCache）与未解码的内存缓存（EncodedMemoryCache）实现**唯一区别**就是已解码内存缓存的数据是CloseableReference<[CloseableBitmap][Closeable]>而未解码内存缓存的数据是CloseableReference<PooledByteBuffer>。即他们的实现方式一样，区别仅仅在于**资源的测量与释放方式不同**。它们使用`ValueDescriptor`来描述不同资源的数据大小，使用不同的`ResourceReleaser`来释放资源。

###1.2.1 LRU缓存载体-CountingHruMap

内存缓存中使用了LRU(Least Recent Used)来提高缓存功能，我们来看一下Fresco中实现它的逻辑。

在`CountingLruMap`中使用了`LinkedHashMap`作为数据存储载体，这个HashMap很特别，它内部有一个双向链表，在做查找操作的时候，从最先插入的单位开始查询。这就提供了一种好处：**它能够很快地删除掉最早插入的单位！**所以它非常适合LRU缓存来使用。

但是由于在`LinkedHashMap`中**重复插入相同单位并不会影响链表顺序**，所以要用`CountingLruMap`将它包装，我们来看看它put对象时的逻辑：

    public synchronized V put(K key, V value) {
        V oldValue = mMap.remove(key);
        mSizeInBytes -= getValueSizeInBytes(oldValue);
        mMap.put(key, value);
        mSizeInBytes += getValueSizeInBytes(value);
        return oldValue;
    }

**它会先将要插入的对象remove掉，然后重新插入该对象！**由此来保证最新加入的对象处于正确地插入顺序中。同时在`mSizeBytes`中更新现在缓存池中所有对象的字节数。

在`CountingHruMap`中还有以下几个重要函数：

- `get(Key)` 查找对象，如有则返回；
- `remove(Key)` 删除对象并返回它；
- `getFirstkey()` 获取最早插入的对象；
- `getCount()` 获取已经缓存的对象数；
- `getSizeInBytes()` 获取缓存池中已经使用的大小。

###1.2.2 具体缓存实现

Fresco中实现具体内存缓存的类是`CountingMemoryCache`，它内部维持着几个重要参数：

- `ExclusiveEntries` 存储着未被使用的对象的`CountingLruMap`；
- `CachedEntries` 存储着所有对象的`CountingLruMap`；
- `MemoryCacheParams` 存储着最大缓存对象数量、缓存池大小等参数、
- `PARAMS_INTERCHECK_INTERVAL_MS` 检查缓存参数变化的事件间隔：5分钟；

它使用一个内部类`Entry`来封装缓存对象，除了记录缓存键、缓存对象之外，它还记录着该对象的引用数量（`clientCount`）及是否被缓存追踪（`isOrphan`）。**注意：每个缓存对象只有满足`clientCount`为0并且`isOrphan`为true时才可以被释放。**（可从`referenceToClose`函数中看出此逻辑）

我们来看看它缓存对象的逻辑：

    public CloseableReference<V> cache(final K key, final CloseableReference<V> valueRef) {
        //检查参数

        maybeUpdateCacheParams();

        CloseableReference<V> oldRefToClose = null;
        CloseableReference<V> clientRef = null;
        synchronized (this) {
            mExclusiveEntries.remove(key);
            Entry<K, V> oldEntry = mCachedEntries.remove(key);
            if (oldEntry != null) {
                makeOrphan(oldEntry);
                oldRefToClose = referenceToClose(oldEntry);
            }

            if (canCacheNewValue(valueRef.get())) {
                Entry<K, V> newEntry = Entry.of(key, valueRef);
                mCachedEntries.put(key, newEntry);
                clientRef = newClientReference(newEntry);
            }
        }
        CloseableReference.closeSafely(oldRefToClose);

        maybeEvictEntries();
        return clientRef;
    }

我们看到它会做几件事：

- 在`maybeUpdateCacheParams`中检查是否需要更新缓存参数；
- 在缓存中查找要插入的对象，若存在则将其移除，并调用它的close方法，如果对象引用数目为0就会被释放；
- 检查是否缓存对象数达到极限或者缓存池被填满，若均为否才能够插入新对象；
- 将插入的对象重新包装进一个[CloseableReference][Closeable]并返回。重新包装对象主要是重设了一下ResourceReleaser，它会在释放资源的时候减少缓存Entry的`clientCount`，并将它加到`ExclusiveEntries`中，如果可释放则直接释放资源。

最后会调用`maybeEvectEntries`函数，判断是否需要释放资源，它的逻辑是：当**ExclusiveEntries**中已经缓存的对象超过缓存池的最大容纳对象**或者**已经超过了缓存池容量时，删除`ExclusiveEntries`中最早插入的对象，直到**ExclusiveEntries**缓存对象小于最大容纳对象**并且**在缓存池容量以内。实际上这个函数会在大部分的缓存操作中出现，保证每次操作结束后缓存处于一个健康的状态。

###1.2.3 Instrument包装

Fresco使用`InstrumentedMemoryCache`包装了`CountingMemoryCache`，主要增加的功能就是提供了`MemoryCacheTracker`，会在缓存命中或未命中时提供回调函数，供使用者实现自定义功能。

###1.2.4 自定义参数

可以通过ImagePipelineConfig的以下两个函数来实现内存缓存参数部分自定义：

- `setBitmapMemoryCacheParamsSupplier(Supplier<MemoryCacheParams> bitmapMemoryCacheParamsSupplier)`
- `setEncodedMemoryCacheParamsSupplier(Supplier<MemoryCacheParams> encodedMemoryCacheParamsSupplier)`

这两个函数都需要提供`MemoryCacheParams`的Supplier，使用者可以自定义ImagePipelineConfig之后在初始化中应用它。

##1.3 文件缓存

由于文件缓存是直接存储在磁盘上的，所以它的实现方式与内存缓存不同，而且带有缓冲区域，所以更加复杂。

###1.3.1 文件存储实现者DiskStorage

`DefaultDiskStorage`负责Fresco中存取文件的逻辑与实现，它是内存与磁盘之间的纽带，具体有以下功能：

- `getResource(String resourceId, Object debugInfo)` 获取描述符指向的文件；
- `contains(String resourceId, Object debugInfo)` 检查是否包含描述符指向的文件；
- `touch(String resourceId, Object debugInfo)` 检查是否包含描述符指向的文件，若包含，则更新该文件被访问的时间；
- `createTemporary(String resourceId, Object debugInfo)` 从描述符指向的文件中提取`BinaryResource`；
- `commit(String resourceId, BinaryResource temporary, Object debugInfo)` 将`BinaryResource`写入描述符指向的文件中；
- `ong remove(String resourceId)` 删除描述符指向的文件，正常返回被删除文件的大小，文件不存在则返回0，其他返回-1。

###1.3.2 DiskStorageSupplier

Fresco使用`DefaultDiskStorageSupplier`提供文件缓存各种底层功能，主要包括以下几个：

- `get()` 返回存储对象，当需要创建或重新创建存储目录时会创建文件目录。

###1.3.2 缓冲区域StagingArea

###1.3.n 自定义参数

可以通过调用`ImagePipelineConfig.setMainDiskCacheConfig(DiskCacheConfig mainDiskCacheConfig)`设置文件缓存。

DiskCacheConfig使用Builder模式创建，它可以自定义缓存目录、缓存文件名、缓存池大小等，具体自定义内容可以参见[DiskCacheConfig](http://fresco-cn.org/javadoc/reference/com/facebook/cache/disk/DiskCacheConfig.html)。

##2 解码

Fresco通过将图片字节码解析成Bitmap并通过将它放在ashmem中来达到一个高效内存应用，在本节中我们将探索它是如何做到的。



[1]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)%20-%20%E5%9B%BE%E5%83%8F%E5%B1%82%E6%AC%A1%E4%B8%8E%E5%90%84%E7%B1%BBDrawable.md
[2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(2)%20-%20GenericDraweeHierarchy%E6%9E%84%E5%BB%BA%E5%9B%BE%E5%B1%82.md
[3]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md
[3-3.2]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[4]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(4)%20-%20%E5%BC%82%E6%AD%A5%E5%8A%A0%E8%BD%BD%E6%95%B0%E6%8D%AE.md
[5]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(5)%20-%20Producer%E6%B5%81%E6%B0%B4%E7%BA%BF.md
[6]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98%E4%B8%8E%E8%A7%A3%E7%A0%81.md

[Supplier]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#supplier
[Closeable]: https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(3)%20-%20DraweeView%E6%98%BE%E7%A4%BA%E5%9B%BE%E5%B1%82%E6%A0%91.md#32-可关闭的引用
[Producer]: https://github.com/desmond1121/Fresco-Source-Analysis/wiki/Fresco%E4%B8%AD%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F#producerconsumer
