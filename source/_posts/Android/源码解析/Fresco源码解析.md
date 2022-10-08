---
title: Fresco源码解析
date: 2020-03-20 15:24:32
tags: 
- Fresco
- 源码解析
categories:
- Android
---

## 1. 介绍：
fresco，facebook开源的针对android应用的图片加载框架，高效和功能齐全。
* 支持加载网络，本地存储和资源图片；
* 提供三级缓存（二级memory和一级internal storage）；
* 支持JPEGs，PNGs，GIFs，WEBPs等，还支持Progressive JPEG，优秀的动画支持；
* 图片圆角，scale，自定义背景，overlays等等；
* 优秀的内存管理, 在安卓低版本中会将缓存保存至特殊区域, 而不是java heap, 从而避免oom；

## 2. 主要组成部分

![fresco基本结构](/images/fresco基本结构.jpg)
* DraweeView：继承于ImageView，只是简单的读取xml文件的一些属性值和做一些初始化的工作，图层管理交由Hierarchy负责，图层数据获取交由ViewHolder负责。
* DraweeHierarchy：由多层Drawable组成，每层Drawable提供某种功能（例如：缩放、圆角）。
* DraweeController：控制数据的获取与图片加载，向pipeline发出请求，并接收相应事件，并根据不同事件控制Hierarchy，从DraweeView接收用户的事件，然后执行取消网络请求、回收资源等操作。
* DraweeHolder：统筹管理Hierarchy与DraweeController。
* ImagePipeline：Fresco的核心模块，用来以各种方式（内存、磁盘、网络等）获取图像。
* Producer/Consumer：Producer也有很多种，是完成具体工作的类. 它用来完成网络数据获取，缓存数据获取、图片解码等多种工作，它产生的结果由Consumer进行消费。
* IO/Data：这一层便是数据层了，负责实现内存缓存、磁盘缓存、网络缓存和其他IO相关的功能。

## 3. 发起图片请求的主要流程
### 3.1 流程图
![fresco发起请求的主要流程](/images/fresco发起请求的流程.jpg)

### 3.2 源码分析

#### 3.2.1 DraweeView
我们常用的类是SimpleDraweeView, 继承关系如下
SimpleDraweeView -> GenericDraweeView -> DraweeView -> ImageView
**注意: 虽然上述的类都是继承于ImageView的, 但是使用时最好不要调用ImageView本身的任何属性和方法, 不然的话使用不了Fresco的功能**

* DraweeView: 持有ViewHolder, ViewHolder管理DraweeController和DraweeHierarchy
* GenericDraweeView: 解析xml属性, 创建DraweeHierarchy
* SimpleDraweeView: 我们一般使用的类, 接受请求, 创建爱你DraweeController

**SimpleDraweeView.setImageURI**
```java
/**
* Displays an image given by the uri.
*
* @param uri uri of the image
* @param callerContext caller context
*/
public void setImageURI(Uri uri, @Nullable Object callerContext) {
DraweeController controller =
    mControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
setController(controller);
}
```
mControllerBuilder在setUri方法中创建了ImageRequest, 在build的过程中构建好了请求, cache, 编码, 解码等流程. 最后setController启动请求流程

#### 3.2.2 DraweeControllerBuilder.build

在DraweeControllerBuilder.build方法中创建了DataSource. 代表数据的来源，用于组装 produce 集合

```
-> AbstractDraweeControllerBuilder.build
--> AbstractDraweeControllerBuilder.buildController
----> PipelineDraweeControllerBuilder.obtainController // 创建controller并return
-----> AbstractDraweeControllerBuilder.obtainDataSourceSupplier
------> AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest // 创建了Supplier<DataSource<IMAGE>>, 调用supplier.get方法就会创建DataSource
-------> PipelineDraweeControllerBuilder.getDataSourceForRequest
--------> ImagePipeline.fetchDecodedImage(...)
```

#### 3.2.3 setController

```
-> DraweeView.setController
--> DraweeHolder.setController
----> DraweeController.setHierarchy
----> DraweeHolder.attachController
-----> AbstractDraweeController.onAttach
------> AbstractDraweeController.submitRequest
```

```java
protected void submitRequest() {
    ...
    final T closeableImage = getCachedImage(); // DataSource还没有start,已经开始获取缓存了
    if (closeableImage != null) {
      ...
      return;
    }
    ...
    mDataSource = getDataSource(); // 获取DataSource
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    // 注册并处理结果
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            ...
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            ...
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
}

@Override
protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
    // 这里的mDataSouceSupplier是controller在创建时有构造方法传入
    DataSource<CloseableReference<CloseableImage>> result = mDataSourceSupplier.get();
    return result;
}
```

还有一个问题, DataSource是什么时候启动的? 我们在DraweeController.build创建过程中发现创建了Supplier<DataSource<>>,  controller的getDataSource实际上就是从Supplier获取的DataSource


```
-------> PipelineDraweeControllerBuilder.getDataSourceForRequest
--------> ImagePipeline.fetchDecodedImage // 在这个方法中创建了producerSequence
---------> ImagePipeline.submitFetchRequest
----------> CloseableProducerToDataSourceAdapter<T>.craete
-----------> new CloseableProducerToDataSourceAdapter
```
**featchDecodeImage**
```java
public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable RequestListener requestListener) {
    try {
      // 创建Producer序列
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext,
          requestListener);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
}
```

**CloseableProducerToDataSourceAdapter的构造方法**
这个构造方法只是简单的调用父类的构造方法
```java
protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener requestListener) {
    
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;

    mRequestListener.onRequestStart(
        settableProducerContext.getImageRequest(),
        mSettableProducerContext.getCallerContext(),
        mSettableProducerContext.getId(),
        mSettableProducerContext.isPrefetch());
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }

    // procuder序列启动
    producer.produceResults(createConsumer(), settableProducerContext);
}
```
**原来DataSource一创建就会启动produer的工作流程**

## 3. Producer序列的工作流程
### 3.1 Producer/Consumer的基本概念
**模板代码**
```java
public class XXXXProducer implements Producer{

    private final Producer mInputProducer;

    public BitmapMemoryCacheProducer(Producer inputProducer) {
        mInputProducer = inputProducer;
    }

    @Override
    public void produceResults(
        final Consumer consumer,
        final ProducerContext producerContext) {

        ... 尝试直接得到结果
        if(已经获取到结果){
            consumer.onNewResult(result, status);
            return ;
        }

        Consumer newConsumer = new DelegatingConsumer(consumer){
            @Override
            public void onNewResultImpl(newResult, int status) {
                ... 处理上一阶段返回的结果
                if(isLast){
                    // 将自己处理完成的数据交给上一层producer
                    // 这里的getConsumer是构造方法传入的consumer, 也就是上一层producer创建的DelegatingConsumer
                    getConsumer().onNewResult();
                }
            }
        }
        // 进行下一阶段
        mInputProducer.produceResults(newConsumer, producerContext);
    }
}
```

**Consumer的onNewResult方法**
onNewResult会直接调用自己的onNewResultImpl方法
```java
@Override
public synchronized void onNewResult(@Nullable T newResult, @Status int status) {
    if (mIsFinished) {
      return;
    }
    mIsFinished = isLast(status);
    try {
      onNewResultImpl(newResult, status);
    } catch (Exception e) {
      onUnhandledException(e);
    }
}
```

按照这样类似责任链的设计模式, 实际上, 往后加入的producer越晚执行

### 3.2 主要的producer内容梳理

* BitmapMemoryCacheGetProducer
从内存中获取已解码的图片, 因为是直接从内存中获取可以用的, 所以是即时的, 在UI线程中就可以做
* BackgroundThreadHandoffProducer
将任务移交给子线程, 这里仅仅是转移线程而已, 具体的工作在具体的线程中完成
* BitmapMemoryCacheKeyMultiplexProducer
将多个拥有相同已解码内存缓存键的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据
* BitmapMemoryCacheProducer
又一次获取内存缓存? 我觉得主要是将下一阶段获取的已解码图片存储到缓存中
* DecodeProducer
解码
* ResizeAndRotateProducer
旋转, 缩放
* AddImageTransformMetaProducer
添加MetaData
* EncodeCacheKeyMutiplexProducer
将多个拥有相同未解码内存缓存键的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据；
* EncodedMemoryCacheProducer
查找未解码的图片缓存, 将下一步得到的未解码图片保存到缓存中
* DiskCacheReadProducer
读磁盘缓存, 有分MainCache和SmallCache, SmallCache存储小图片, 避免大图片被挤出缓存. 启动task并扔进线程池
* DiskCacheWriteProducer
存入磁盘缓存, 同样是在线程池中操作
* newNetworkFetchProducer
从网络中获取图片

## 4. Fresco 在内存管理上的优势

  - 在Dalvik虚拟机中，gc性能较差，会伴有stop-the-world的发生，导致卡顿，所以Fresco会将解码之后的Bitmap存放到Ashmem当中，并且每次解码完都会通过Native层的代码进行PinBitmap的操作，防止被系统回收。 Fresco使用了 CloseableReference 进行引用计数，手动回收bitmap对象。
  - 在Art虚拟机中，gc性能得到了大幅的提升，所以没必要用各种骚操作，直接将Bitmap解码到Java堆当中即可。

### 4.1 CloseableReference 引用计数

[使用 CloseableReference 优雅的释放对象，来自 Fresco](https://juejin.cn/post/7026540580788240415#heading-3)

CloseableReference 是 fresco 内部建立的一个机制，核心思想是 用static的集合屏蔽java的gc的内存回收，转变为类似c/c++的手动释放内存。
在Dalvik虚拟机中，gc性能较差，bitmap的频繁gc会影响性能

```java
public final class CloseableReference<T> implements Cloneable, Closeable {

// 构造方法是private，只能使用 of(...) 方法进行实例化
  private CloseableReference(T t, ResourceReleaser<T> resourceReleaser) {
    mSharedReference = new SharedReference<T>(t, resourceReleaser);
  }

  @Override
  public void close() {
    synchronized (this) {
      if (mIsClosed) {
        return;
      }
      mIsClosed = true;
    }

    mSharedReference.deleteReference();
  }

// clone 就是把引用计数+1
  public synchronized CloseableReference<T> clone() {
    return new CloseableReference<T>(mSharedReference);
  }

  private CloseableReference(SharedReference<T> sharedReference) {
    mSharedReference = Preconditions.checkNotNull(sharedReference);
    sharedReference.addReference();
  }
}

public class SharedReference<T> {

  public static Map<Object, Integer> getLiveObjects() {
    return sLiveObjects;
  }

  public SharedReference(T value, ResourceReleaser<T> resourceReleaser) {
    mValue = Preconditions.checkNotNull(value);
    mResourceReleaser = Preconditions.checkNotNull(resourceReleaser);
    mRefCount = 1;
    // 添加进 static 的集合中，规避jvm的内存回收
    addLiveReference(value);
  }

  private static void addLiveReference(Object value) {
    synchronized (sLiveObjects) {
      Integer count = sLiveObjects.get(value);
      if (count == null) {
        sLiveObjects.put(value, 1);
      } else {
        sLiveObjects.put(value, count + 1);
      }
    }
  }

  // 引用计数+1
  public synchronized void addReference() {
    mRefCount++;
  }

  // 引用计数-1
  public void deleteReference() {
    if (decreaseRefCount() == 0) {
      T deleted;
      synchronized (this) {
        deleted = mValue;
        mValue = null;
      }
      mResourceReleaser.release(deleted);
      removeLiveReference(deleted);
    }
  }
  
  public synchronized int decreaseRefCount() {
    mRefCount--;
    return mRefCount;
  }
}
```

### 4.2 CloseableReference 在 LruCache 的使用

CountingMemoryCache

```java
public class 65<K, V> implements MemoryCache<K, V>, MemoryTrimmable {

  /**
  * 有两个集合， mExclusiveEntries 等待移除的entry，引用为0的entry的集合
  * mCachedEntries 是缓存
  **/
  final CountingLruMap<K, Entry<K, V>> mExclusiveEntries;
  final CountingLruMap<K, Entry<K, V>> mCachedEntries;

  @Nullable
  public CloseableReference<V> get(final K key) {
    Entry<K, V> oldExclusive;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      // 从 mExclusiveEntries ，引用会增加，一定不会回收
      oldExclusive = mExclusiveEntries.remove(key);
      Entry<K, V> entry = mCachedEntries.get(key);
      if (entry != null) {
        // 重要
        clientRef = newClientReference(entry);
      }
    }
    maybeNotifyExclusiveEntryRemoval(oldExclusive);
    maybeUpdateCacheParams();
    // 尝试回收 mExclusiveEntries 的元素
    maybeEvictEntries();
    return clientRef;
  }

  public CloseableReference<V> cache(
      final K key,
      final CloseableReference<V> valueRef,
      final EntryStateObserver<K> observer
  ) {

    maybeUpdateCacheParams();

    Entry<K, V> oldExclusive;
    CloseableReference<V> oldRefToClose = null;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      oldExclusive = mExclusiveEntries.remove(key);
      // 从lrucache中找到得到旧的entry
      Entry<K, V> oldEntry = mCachedEntries.remove(key);
      if (oldEntry != null) {
        makeOrphan(oldEntry);
        oldRefToClose = referenceToClose(oldEntry);
      }

      if (canCacheNewValue(valueRef.get())) {
        Entry<K, V> newEntry = Entry.of(key, valueRef, observer);
        // 新entry加入集合
        mCachedEntries.put(key, newEntry);
        clientRef = newClientReference(newEntry);
      }
    }
    // 旧entry引用计数-1
    CloseableReference.closeSafely(oldRefToClose);
    maybeNotifyExclusiveEntryRemoval(oldExclusive);

    // 尝试回收
    maybeEvictEntries();
    return clientRef;
  }

  private synchronized CloseableReference<V> newClientReference(final Entry<K, V> entry) {
    increaseClientCount(entry);
    return CloseableReference.of(
        entry.valueRef.get(),
        new ResourceReleaser<V>() {
          @Override
          public void release(V unused) {
            // 当有一次引用释放发生时，会回调这里
            releaseClientReference(entry);
          }
        });
  }

  private void releaseClientReference(final Entry<K, V> entry) {
    Preconditions.checkNotNull(entry);
    boolean isExclusiveAdded;
    CloseableReference<V> oldRefToClose;
    synchronized (this) {
      decreaseClientCount(entry);
      isExclusiveAdded = maybeAddToExclusives(entry);
      oldRefToClose = referenceToClose(entry);
    }
    // 引用计数-1
    CloseableReference.closeSafely(oldRefToClose);
    maybeNotifyExclusiveEntryInsertion(isExclusiveAdded ? entry : null);
    maybeUpdateCacheParams();
    // 尝试回收
    maybeEvictEntries();
  }
}
```

这个cache就是将 LruCache 和 引用计数的思想结合

### 4.3 针对 Dalvik 虚拟机的缓存优化

在 DecodeProducer 中，进行了图片的解码。 最终调用解码是在 `ProgressiveDecoder#doDecode` 方法中

```java
    private void doDecode(EncodedImage encodedImage, @Status int status) {
      ...
      image = mImageDecoder.decode(encodedImage, length, quality, mImageDecodeOptions);
      ...
    }
```

mImageDecoder的实现类根据系统版本有所不同，在 api21 也就是 art虚拟机实现类是 `DefaultDecoder` 。 在 Dalvik虚拟机实现类是 `DalvikPurgeableDecoder`

以jpg图片的解码举例 
```java
  @Override
  public CloseableReference<Bitmap> decodeJPEGFromEncodedImageWithColorSpace(
      final EncodedImage encodedImage,
      Bitmap.Config bitmapConfig,
      @Nullable Rect regionToDecode,
      int length,
      final boolean transformToSRGB) {
    BitmapFactory.Options options = getBitmapFactoryOptions(
        encodedImage.getSampleSize(),
        bitmapConfig);
    final CloseableReference<PooledByteBuffer> bytesRef = encodedImage.getByteBufferRef();
    Preconditions.checkNotNull(bytesRef);
    try {
      Bitmap bitmap = decodeJPEGByteArrayAsPurgeable(bytesRef, length, options);
      return pinBitmap(bitmap);
    } finally {
      CloseableReference.closeSafely(bytesRef);
    }
  }
```

这里做了两个操作，先解码得到bitmap，然后对pin这个bitmap。 pin的作用是让的bitmap对象避免被系统gc

先来看一下解码
```java

  @Override
  protected Bitmap decodeJPEGByteArrayAsPurgeable(
      CloseableReference<PooledByteBuffer> bytesRef, int length, BitmapFactory.Options options) {
    byte[] suffix = endsWithEOI(bytesRef, length) ? null : EOI;
    return decodeFileDescriptorAsPurgeable(bytesRef, length, suffix, options);
  }

  private Bitmap decodeFileDescriptorAsPurgeable(
      CloseableReference<PooledByteBuffer> bytesRef,
      int inputLength,
      byte[] suffix,
      BitmapFactory.Options options) {
    MemoryFile memoryFile = null;
    try {
      memoryFile = copyToMemoryFile(bytesRef, inputLength, suffix);
      FileDescriptor fd = getMemoryFileDescriptor(memoryFile);
      if (mWebpBitmapFactory != null) {
        Bitmap bitmap = mWebpBitmapFactory.decodeFileDescriptor(fd, null, options);
        return Preconditions.checkNotNull(bitmap, "BitmapFactory returned null");
      } else {
        throw new IllegalStateException("WebpBitmapFactory is null");
      }
    } catch (IOException e) {
      throw Throwables.propagate(e);
    } finally {
      if (memoryFile != null) {
        memoryFile.close();
      }
    }
  }
```

这里有一个 MemoryFile ， 这是安卓系统提供的用来使用匿名共享内存的类(ashmem) , 也就是说在 Dalvik 会将bitmap保存到 ashmem 里，避免被触碰到java层的oom限制。

然后是 pin
```java
  public CloseableReference<Bitmap> pinBitmap(Bitmap bitmap) {
    // jni调用pin
    nativePinBitmap(bitmap);
    // 使用 CloseableReference 。最终会在引用计数为0时，手动触发回收
    return CloseableReference.of(bitmap, mUnpooledBitmapsCounter.getReleaser());
  }

/**
      mUnpooledBitmapsReleaser = new ResourceReleaser<Bitmap>() {
      @Override
      public void release(Bitmap value) {
        try {
          decrease(value);
        } finally {
          value.recycle();
        }
      }
    };
**/
```

### Dalvik 和 Art 在gc上的差异
[揭秘 ART 细节 ---- Garbage collection](https://www.cnblogs.com/jinkeep/p/3818180.html)

1. art 新增了 large object space ，专供大对象使用
2. Dalvik 在垃圾回收时，会暂停所有线程，在内存紧张时会频繁执行，容易造成卡顿丢帧。art优化了回收算法，会减少暂停线程的次数
   
  





