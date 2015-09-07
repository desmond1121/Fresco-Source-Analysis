#Fresco源码分析(6) - 缓存
---

> 作者：[Desmond 转载请注明出处！](https://github.com/desmond1121)

Fresco中一共有三级缓存，其中前两级内存缓存都存储在Java堆上，本地缓存存储在本地文件目录中。

##1 CacheKey

Fresco中专门用于缓存键的接口，有这两种类实现了`CacheKey`：

- `BitmapMemoryCacheKey` 用于已解码的内存缓存键，会对Uri字符串、缩放尺寸、解码参数、PostProcessor等关键参数进行hashCode作为唯一标识；
- `SimpleCacheKey` 普通的缓存键实现，使用传入字符串的hashCode作为唯一标识，所以需要保证相同键传入字符串相同。

##2 内存缓存

已解码的内存缓存（BitmapMemoryCache）与未解码的内存缓存（EncodedMemoryCache）实现**唯一区别**就是已解码内存缓存的数据是CloseableReference<[CloseableBitmap][Closeable]>而未解码内存缓存的数据是CloseableReference<PooledByteBuffer>。即他们的实现方式一样，区别仅仅在于**资源的测量与释放方式不同**。它们使用`ValueDescriptor`来描述不同资源的数据大小，使用不同的`ResourceReleaser`来释放资源。

###2.1 LRU缓存载体-CountingLruMap

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

###2.2 具体缓存实现

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

###2.3 Instrument包装

Fresco使用`InstrumentedMemoryCache`包装了`CountingMemoryCache`，主要增加的功能就是提供了`MemoryCacheTracker`，会在缓存命中或未命中时提供回调函数，供使用者实现自定义功能。

###2.4 自定义参数

可以通过ImagePipelineConfig的以下两个函数来实现内存缓存参数部分自定义：

- `setBitmapMemoryCacheParamsSupplier(Supplier<MemoryCacheParams> bitmapMemoryCacheParamsSupplier)`
- `setEncodedMemoryCacheParamsSupplier(Supplier<MemoryCacheParams> encodedMemoryCacheParamsSupplier)`

这两个函数都需要提供`MemoryCacheParams`的Supplier，使用者可以自定义ImagePipelineConfig之后在初始化中应用它。

##3 文件缓存

由于文件缓存是直接存储在磁盘上的，所以它的实现方式与内存缓存不同，而且带有缓冲区域，所以更加复杂。我总结它一共有三层内容：文件存储层，文件缓存层，缓冲缓存层。

###3.1 文件存储层

文件缓存都是将实际的文件存储在存储设备中，Fresco的文件存储有两种格式的文件：

1. `.cnt` 实际存储的内容文件
2. `.tmp` 临时文件

Fresco中定义了`BinaryResource`来封装文件对象，你可以通过它获取文件的输入流、字节码等。此外，Fresco定义了每个文件的唯一描述符，此描述符由`CacheKey`的`.toString()`导出字符串的SHA-1哈希码再经过Base64加密得出。

文件存储层有两个重要工具：DefaultDiskStorageSupplier与DefaultDiskStorage。

`DefaultDiskStorageSupplier`可以用来创建缓存文件目录及获取到对应的`DefaultDiskStorage`，是一个典型的[Supplier][Supplier]。

`DefaultDiskStorage`是文件操作者，它实现了`DiskStorage`接口，负责Fresco中存取文件的逻辑与实现，具体有以下功能：

- `getEntries()` 获取缓存目录下所有文件的`Entry`（Entry对象中存储着文件的BinaryResource、TimeStamp及大小）。
- `getResource(String resourceId, Object debugInfo)` 获取描述符指向的文件，更新时间戳；
- `contains(String resourceId, Object debugInfo)` 检查是否包含描述符指向的文件；
- `createTemporary(String resourceId, Object debugInfo)` 以指定描述符创建临时文件；
- `commit(String resourceId, BinaryResource temporary, Object debugInfo)` 将`BinaryResource`写入描述符指向的文件中，更新时间戳；
- `remove(String resourceId)` 删除描述符指向的文件，正常返回被删除文件的大小，文件不存在则返回0，其他返回-1。

###3.2 文件缓存层

DiskStorageCache是Fresco实现文件缓存的主要类，在文件缓存中也使用了相应的LRU技术提高缓存效率，我们来看看它是怎么实现的。

####3.2.1 LRU实现

在evictAboveSize中我们可以看到所使用的LRU逻辑：

    private void evictAboveSize(
            long desiredSize,
            CacheEventListener.EvictionReason reason) throws IOException {
        DiskStorage storage = mStorageSupplier.get();
        Collection<DiskStorage.Entry> entries;
        try {
            entries = getSortedEntries(storage.getEntries());
        } catch (IOException ioe) {
            //异常捕捉
        }

        //要删除的数据量
        long deleteSize = mCacheStats.getSize() - desiredSize;

        //记录删除数据数量
        int itemCount = 0;

        //记录删除数据的大小
        long sumItemSizes = 0L;

        for (DiskStorage.Entry entry : entries) {
            if (sumItemSizes > (deleteSize)) {
                break;
            }
            long deletedSize = storage.remove(entry);
            if (deletedSize > 0) {
                itemCount++;
                sumItemSizes += deletedSize;
            }
        }
        mCacheStats.increment(-sumItemSizes, -itemCount);
        storage.purgeUnexpectedResources();
        reportEviction(reason, itemCount, sumItemSizes);
    }

我们可以看到在这个函数中它获取两个输入：期望达到的剩余容量及缓存事件监听者。接下来以以下顺序进行操作：

1. 获取存储着缓存目录下所有文件`Entry`的Collection，以它们被访问时间进行排序，最近被访问的Entry在后面；
2. 从Collection的头部开始删除文件，**直到剩余容量达到desiredSize为止**；
3. 更新容量，删除不需要的文件（临时文件等），汇报缓存工作。

在`maybeEvictFilesInCacheDir`函数中我们可以看到当缓存过载时会**以缓存容量的90%为目标**进行清理。

####3.2.2 同步操作

在文件缓存中维持一个对象`mLock`，该对象就是为了让各个操作保持同步。我们来看一下`DiskStorageCache`中的几个重要底层操作函数及它们的同步情况：

- `getResource(final CacheKey key)` **同步操作**，从指定CacheKey获取文件描述符，如果存在则返回它的`BinaryResource`；
- `createTemporaryResource(String resourceId, CacheKey key)` **非同步操作**，检查是否需要清理缓存，如果需要则进行清理，之后创建临时文件并返回它的BinaryResource；
- `deleteTemporaryResource(BinaryResource temporaryResource)` **非同步操作**，删除BinaryResource指向的文件；
- `BinaryResource commitResource(String resourceId, CacheKey key, BinaryResource temporary)` **同步操作**，将temporary写入文件描述符指向的文件；

这么做主要是要保证写入最终缓存文件的原子性，我们可以看它提供的写入缓存函数:

    public BinaryResource insert(CacheKey key, WriterCallback callback) throws IOException {
        mCacheEventListener.onWriteAttempt();
        final String resourceId = getResourceId(key);
        try {
            BinaryResource temporary = createTemporaryResource(resourceId, key);
            try {
                mStorageSupplier.get().updateResource(resourceId, temporary, callback, key);
                return commitResource(resourceId, key, temporary);
            } finally {
                deleteTemporaryResource(temporary);
            }
        } catch (IOException ioe) {
            //异常处理
        }
    }

我们看到它在插入的时候会首先创建临时文件（此时多个任务可以并行地操作），而在将temporary数据commit到描述符指定的文件中是同步操作。

**注意：这个函数并没有直接提供要写入的数据**，而是在`updateResource`函数中通过`WriterCallback`实现的自定义写入函数将数据写到temporary中，会在[缓冲缓存层](https://github.com/desmond1121/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(6)%20-%20%E7%BC%93%E5%AD%98%E4%B8%8E%E8%A7%A3%E7%A0%81.md#133-缓冲缓存层)中进一步解释。

此外，它的删除资源操作`remove`函数也是同步的。

###3.3 缓冲缓存层

缓冲缓存层BufferedDiskCache将缓存层进行包装，它主要多了三个功能：

1. 提供写入缓冲StagingArea，所有要写入的数据在发出写入命令到最终写入之前会存储在这里，在查找缓存的时候会首先在这块区域内查找，若命中则直接返回；
2. 提供了写入数据的办法，在`writeToDiskCache`中可以看出它提供的`WriterCallback`将要写入的`EncodedImage`转码成输入流；
3. 将get、put两个方法放在后台线程中运行（get时在缓冲区域查找时除外），分别都是容量为2的线程池。

###3.4 自定义参数

可以通过调用`ImagePipelineConfig.setMainDiskCacheConfig(DiskCacheConfig mainDiskCacheConfig)`设置文件缓存。

DiskCacheConfig使用Builder模式创建，它可以自定义缓存路径、缓存文件夹名称、缓存池大小等，具体自定义内容可以参见[DiskCacheConfig](http://fresco-cn.org/javadoc/reference/com/facebook/cache/disk/DiskCacheConfig.html)。


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
