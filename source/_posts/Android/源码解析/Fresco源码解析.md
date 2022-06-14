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

在DraweeControllerBuilder.build方法中创建了DataSource. DataSource类似于Java里的Futures，代表数据的来源

```
-> AbstractDraweeControllerBuilder.build
--> AbstractDraweeControllerBuilder.buildController
----> PipelineDraweeControllerBuilder.obtainController // 创建controller并return
-----> AbstractDraweeControllerBuilder.obtainDataSourceSupplier
------> AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest // 创建了Supplier<DataSource<IMAGE>>, 调用supplier.get方法就会创建Data
Source
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

