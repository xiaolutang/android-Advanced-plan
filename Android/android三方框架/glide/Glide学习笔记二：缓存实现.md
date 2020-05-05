



在上一篇学习笔记

[Glide加载流程分析]: https://juejin.im/post/5eaad7e1f265da7bd802a819

中我们大致了解了Glide的加载流程，这里我们主要分析一下Glide的缓存。

Glide的缓存可以说考虑的非常全面分为下面的几个方面

内存加载

- 活动资源加载。如果加载成功，那么引用计数加一
- LRU内存缓存加载。如果加载成功，那么从LRU缓存中移除，并且将对应的数据放入到活动资源。

文件加载（如果前面两个流程执行成功就不会执行后面的）

- 从文件活动资源中加载
- 从文件中加载
- 从网络中加载
- 缓存 到文件中
- 缓存到文件活动资源
- 将数据放在活动资源中

文章主要致力于Glide四级缓存的加载过程的梳理，而不会过渡执着于详细的加载过程。

# Glide缓存Key

Glide在Engine类的load方法中生成了缓存Key，它的代码如下

```java
EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,    resourceClass, transcodeClass, options);
```

keyFactory为EngineKeyFactory,在它的build方法中通过new的方式创建了EngineKey。而EngineKey主要是重写了equals和hashCode方法，简单的来理解就是各个参数都相同的时候我们就认为他们是同一个key。

# Glide的内存缓存

​	内存缓存分为两个部分，活动资源缓存，LRU缓存

## 活动资源缓存

加载活动资源缓存的代码如下：

```java
EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
```

```java
private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }

    return active;
  }
```

活动资源缓存采用引用计数的方式判断当前资源的使用次数，每加从活动资源中加载一次计数器加一，每次回收减一。当引用计数器为0的时候会将活动资源中的缓存放回LRU缓存。

### LRU缓存

LRU缓存简单的来说就是最近最少使用原则。

从LRU缓存中读取图片的代码如下：

```
EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
```

```java
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
  }
```

可以看到当从Lru中读取到数据后，会将数据放入到活动资源。并将它的引用计数器加一

细心的朋友可能注意到了，不论是从活动资源还是LRU缓存中加载都有一个参数isMemoryCacheable。它表示是否使用内存缓存。一旦为false。内存缓存就不会生效。可以在图片加载的时候通过skipMemoryCache进行配置。

```
Glide.with(this).load("").skipMemoryCache(true).into(iamgeView)
```

# 文件缓存

Glide选择磁盘缓存的方式如下：

```
Glide.with(this)
     .load(url)
     .diskCacheStrategy(DiskCacheStrategy.ALL)
     .into(imageView);
```

diskCacheStrategy有5种实现方式

DiskCacheStrategy.NONE： 表示不缓存任何内容。
DiskCacheStrategy.DATA： 表示只缓存原始图片。
DiskCacheStrategy.RESOURCE： 表示只缓存转换过后的图片。
DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。AUTOMATICDiskCacheStrategy.AUTOMATIC： Glide自动选择缓存策略（默认选项）。

当图片在内存缓存中加载失败之后就会在文件缓存中进行查找。这一部分加载主要在DecodeJob进行。DecodeJob实现了Runnable。仔细分析后我发现这个类是一个很有意思的。这里为了了解整个加载的全过程。我们设置缓存策略为DiskCacheStrategy.ALL

这里在回顾下DiskCacheStrategy 它有4个方法

```
public abstract class DiskCacheStrategy {

  

 //是否允许缓存原始图片
  public abstract boolean isDataCacheable(DataSource dataSource);

 //是否允许缓存指定大小的图片
  public abstract boolean isResourceCacheable(boolean isFromAlternateCacheKey,
      DataSource dataSource, EncodeStrategy encodeStrategy);

  //是否读取指定大小的图片
  public abstract boolean decodeCachedResource();

  //是否读取原始图片
  public abstract boolean decodeCachedData();
}

```

## DecodeJob的工作过程

DecodeJob的实现了Runnable接口它的工作开始于run方法。而run方法的核心调用逻辑在runWrapped中实现

在DecodeJob创建的时候stage初始状态DecodeJob#Stage#INITIALIZE。在runWrapped中获取getNextStage返回的state为DecodeJob#Stage#RESOURCE_CACHE。getNextGenerator返回的是ResourceCacheGenerator。

```java
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE://从缓存中加载
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE://加载网络数据，并缓存
        runGenerators();
        break;
      case DECODE_DATA://从文件中加载数据
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```

```java

  private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE://在initialize的情况下，如果允许从资源缓存中读取数据返回状态RESOURCE_CACHE，否则获取下一个状态
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE://如果允许从原数据中读取，就返回DATA_CACHE，否则读取下一个状态。
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE://如果只是从缓存中读取，则返回状态FINISHEd否则返回SOURCE
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```

```java
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```

在第一次执行这个方法的时候会执行先从ResourceCacheGenerator中获取数据，如何ResourceCacheGenerator#startNext返回false。会获取下一状态的currentGenerator，为DataCacheGenerator。并且执行它的startNext方法。

```
private void runGenerators() {
    ...
    while (!isCancelled && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
   ...
  }
```

如果当前状态为SOURCE时，会调用reschedule（），并退出当前循环

```java
@Override
  public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
  }
```

可以看到reschedule将runReason设置为SWITCH_TO_SOURCE_SERVICE，并重新执行run方法。此时会执行SourceGenerator#startNext方法从网络中加载数据。当数据加载成功的时候会调用到SourceGenerator#onDataReady。

```java
  public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
          loadData.fetcher.getDataSource(), originalKey);
    }
  }
```

如果配置允许文件缓存。则会再次重新执行DecodeJob#run方法。因为前面执行过一次数据加载。dataToCache不为空，这一次不会进行数据加载，而是数据缓存。

```java
@Override
  public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
          || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }
```

在cacheData中会执行文件缓存和 构建DataCacheGenerator从文件中读取数据。

```java
private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
      Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
      DataCacheWriter<Object> writer =
          new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
      originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      helper.getDiskCache().put(originalKey, writer);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Finished encoding source to cache"
            + ", key: " + originalKey
            + ", data: " + dataToCache
            + ", encoder: " + encoder
            + ", duration: " + LogTime.getElapsedMillis(startTime));
      }
    } finally {
      loadData.fetcher.cleanup();
    }

    sourceCacheGenerator =
        new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
  }
  
```

在文件数据加载完成后会调用到DecodeJob#onDataFetcherReady进行图片数据加载和进行对应的变换。并且在DecodeJob#decodeFromRetrievedData中调用DecodeJob#decodeFromData进行数据加载。在DecodeJob#notifyEncodeAndRelease进行数据缓存到内存活动资源。这样就完成了一次图片的全部流程。



# 总结：

图片的加载过程分为两个部分，

内存加载

- 活动资源加载。如果加载成功，那么引用计数加一
- LRU内存缓存加载。如果加载成功，那么从LRU缓存中移除，并且将对应的数据放入到活动资源。

文件加载（如果前面两个流程执行成功就不会执行后面的）

- 从文件活动资源中加载
- 从文件中加载
- 从网络中加载
- 缓存 到文件中
- 缓存到文件活动资源
- 将数据放在活动资源中



