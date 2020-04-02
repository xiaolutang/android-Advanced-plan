

这一篇是在上一篇的基础上加深我们对glide的理解，如果您对glide还没有一个感性的认识建议看一下。



这篇主要回答上篇文章的几个问题：

# RequestManager如何与Activity&Fragment进行生命周期的绑定

由上一篇文章我们知道RequestManager实际上是由RequestManagerRetriever来进行获取的。当传入对象是Fragment时

```java
@NonNull
  public RequestManager get(@NonNull Fragment fragment) {
    Preconditions.checkNotNull(fragment.getActivity(),
          "You cannot start a load on a fragment before it is attached or after it is destroyed");
    if (Util.isOnBackgroundThread()) {
      return get(fragment.getActivity().getApplicationContext());
    } else {
      FragmentManager fm = fragment.getChildFragmentManager();
      return supportFragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
    }
  }
```

supportFragmentGet方法：

```java
@NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```

可以看到supportFragmentGet方法首先通过传入Fragment获取的FragmentManager来获取一个

SupportRequestManagerFragment对象，如果这个对象的RequestManager为空就创建一个，并且将它设置个这个SupportRequestManagerFragment。

接下来我们来看看SupportRequestManagerFragment是如何创建的

```java
@NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```

上面的代码的意思是：给当前的fragment添加一个tag为FRAGMENT_TAG并且空白透明的子fragment,并且根据这个空白透明fragmnet来对这个requestManager进行生命周期管理，这样fragment就和requestManager进行生命周期关联。

类似的当传入的对象是activity的时候就给它设置透明的fragment,将activity和requestmanager进行生命周期的关联。

# glide的加载&缓存如何实现

我们知道Glide的正真开始处理图片加载相关开始于Engine的load方法。

图片的加载顺序是：

1. 活动资源
2. lru内存缓存
3. 资源类型
4. 数据来源

相关的分析直接注释在代码中：

```
public synchronized <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);
//从活动资源中进行加载
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }
//从lru缓存中进行加载
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
//当活动资源和lru内存缓存都不存在的时候从DecodeJob中进行加载
//DecodeJob实际上是一个Runnable,对应的处理开始于其run方法
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }
```

DecodeJob相关代码：

DecodeJob的run方法会执行runWrapped

```java
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }

private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }

  private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }

    // Otherwise a generator started a new load and we expect to be called back in
    // onDataFetcherReady.
  }
```

DecodeJob的6中状态

RESOURCE_CACHE对应从资源缓存中进行加载

DATA_CACHE对象从数据来源中进行加载

SOURCE对应于从网络中进行加载

```java
private enum Stage {
    /** The initial stage. */
    INITIALIZE,
    /** Decode from a cached resource. */
    RESOURCE_CACHE,
    /** Decode from cached source data. */
    DATA_CACHE,
    /** Decode from retrieved source. */
    SOURCE,
    /** Encoding transformed resources after a successful load. */
    ENCODE,
    /** No more viable stages. */
    FINISHED,
  }
```

在默认情况下他们从初始状态到结束状态依次调用的。

在分析ResourceCacheGenerato的startNext之前为了能够更好的理解glide的加载过程我们需要补充一点知识。Glide的ModelLoader<Model, Data>，我们知道如果requestManager传入的是网络地址那么进行网络加载的是HttpUrlFetcher（上一篇文章debug发现的）。那么它是如何来呢？这里我们对它的原理进行说明。

在glide类的构造方法中会注册一些列的ModelLoader。这些注册的ModelLoader最终被ModelLoaderRegistry进行管理。由于这个代码较多我们只关注网络的图片加载，其它的可以按照类似的原理进行分析。

先来看看Glide中注册的是String类型的ModelLoader

```java
Glide(){
//省略代码
 .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
        .append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
        .append(String.class, InputStream.class, new StringLoader.StreamFactory())
        .append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
        .append(
            String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory());
//省略代码
}
```

在忽略其他代码的情况下我们可以看到 ModelLoader<Model, Data>的Model为String类型的有4种，惊讶的发现这里注册的居然没有网络类型的ModelLoader。不过不必心急我们慢慢分析。

ModelLoader定义了一个handles方法返回表示是否能够处理对应的model。我们发现只有StringLoader能够处理网络地址的请求。来看看它的实现

```java
public class StringLoader<Data> implements ModelLoader<String, Data> {
  private final ModelLoader<Uri, Data> uriLoader;

  // Public API.
  @SuppressWarnings("WeakerAccess")
  public StringLoader(ModelLoader<Uri, Data> uriLoader) {
    this.uriLoader = uriLoader;
  }

  @Override
  public LoadData<Data> buildLoadData(@NonNull String model, int width, int height,
      @NonNull Options options) {
    Uri uri = parseUri(model);
    if (uri == null || !uriLoader.handles(uri)) {
      return null;
    }
     //将创建LoadData的工作交给uriLoader
    return uriLoader.buildLoadData(uri, width, height, options);
  }

  @Override
  public boolean handles(@NonNull String model) {
    // Avoid parsing the Uri twice and simply return null from buildLoadData if we don't handle this
    // particular Uri type.
    return true;
  }

  @Nullable
  private static Uri parseUri(String model) {
    Uri uri;
    if (TextUtils.isEmpty(model)) {
      return null;
    // See https://pmd.github.io/pmd-6.0.0/pmd_rules_java_performance.html#simplifystartswith
    } else if (model.charAt(0) == '/') {
      uri = toFileUri(model);
    } else {
      uri = Uri.parse(model);
      String scheme = uri.getScheme();
      if (scheme == null) {
        uri = toFileUri(model);
      }
    }
    return uri;
  }

  private static Uri toFileUri(String path) {
    return Uri.fromFile(new File(path));
  }

  /**
   * Factory for loading {@link InputStream}s from Strings.
   */
  public static class StreamFactory implements ModelLoaderFactory<String, InputStream> {

    @NonNull
    @Override
    public ModelLoader<String, InputStream> build(
        @NonNull MultiModelLoaderFactory multiFactory) {
      return new StringLoader<>(multiFactory.build(Uri.class, InputStream.class));
    }

    @Override
    public void teardown() {
      // Do nothing.
    }
  }

  /**
   * Factory for loading {@link ParcelFileDescriptor}s from Strings.
   */
  public static class FileDescriptorFactory
      implements ModelLoaderFactory<String, ParcelFileDescriptor> {

    @NonNull
    @Override
    public ModelLoader<String, ParcelFileDescriptor> build(
        @NonNull MultiModelLoaderFactory multiFactory) {
      return new StringLoader<>(multiFactory.build(Uri.class, ParcelFileDescriptor.class));
    }

    @Override
    public void teardown() {
      // Do nothing.
    }
  }

  /**
   * Loads {@link AssetFileDescriptor}s from Strings.
   */
  public static final class AssetFileDescriptorFactory
      implements ModelLoaderFactory<String, AssetFileDescriptor> {

    @Override
    public ModelLoader<String, AssetFileDescriptor> build(
        @NonNull MultiModelLoaderFactory multiFactory) {
      return new StringLoader<>(multiFactory.build(Uri.class, AssetFileDescriptor.class));
    }

    @Override
    public void teardown() {
      // Do nothing.
    }
  }
}
```

可以看到在它的buildLoadData方法中将创建lodaData的工作交给了uriLoader，而uriLoader是在构造方法中传递进来的。通过源码我们知道StringLoader的创建只有在它的静态内部类中的几个工厂类里面，从它们的创建中我们可以知道。具体的创建uriLoader都交给了MultiModelLoaderFactory。并且model都是URI。

这样我们知道当传入的是一个String类型的model的时候因为DataUrlLoader只能处理以data:image开始的字符串。我们能够获取三个StringLoader。然后这三种类型又通过一定的转换方式将处理交给model为Uri，data分别为InputStream,ParcelFileDescriptor,AssetFileDescriptor的modelLoader。

同样的我们在Glide的构造方法中去查看它们的注册分别是：

```java
.append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
.append(Uri.class, InputStream.class, new HttpUriLoader.Factory())
        .append(Uri.class, InputStream.class, new AssetUriLoader.StreamFactory(context.getAssets()))
        .append(
            Uri.class,
            ParcelFileDescriptor.class,
            new AssetUriLoader.FileDescriptorFactory(context.getAssets()))
.append(Uri.class, InputStream.class, new MediaStoreImageThumbLoader.Factory(context))
        .append(Uri.class, InputStream.class, new MediaStoreVideoThumbLoader.Factory(context))
        .append(
            Uri.class,
            InputStream.class,
            new UriLoader.StreamFactory(contentResolver))
        .append(
            Uri.class,
            ParcelFileDescriptor.class,
             new UriLoader.FileDescriptorFactory(contentResolver))
        .append(
            Uri.class,
            AssetFileDescriptor.class,
            new UriLoader.AssetFileDescriptorFactory(contentResolver))
        .append(Uri.class, InputStream.class, new UrlUriLoader.StreamFactory())
```

而在这个里面能够处理网络加载的只有HttpUriLoader而他的网络加载处理又交给model为GlideUrl的modelLoader来进行处理，Glide中注册model为GlideUrl的只有HttpGlideUrlLoader。最终进行网络加载的loadData如下

```java
public class HttpGlideUrlLoader implements ModelLoader<GlideUrl, InputStream> {
  /**
   * An integer option that is used to determine the maximum connect and read timeout durations (in
   * milliseconds) for network connections.
   *
   * <p>Defaults to 2500ms.
   */
  public static final Option<Integer> TIMEOUT = Option.memory(
      "com.bumptech.glide.load.model.stream.HttpGlideUrlLoader.Timeout", 2500);

  @Nullable private final ModelCache<GlideUrl, GlideUrl> modelCache;

  public HttpGlideUrlLoader() {
    this(null);
  }

  public HttpGlideUrlLoader(@Nullable ModelCache<GlideUrl, GlideUrl> modelCache) {
    this.modelCache = modelCache;
  }

  @Override
  public LoadData<InputStream> buildLoadData(@NonNull GlideUrl model, int width, int height,
      @NonNull Options options) {
    // GlideUrls memoize parsed URLs so caching them saves a few object instantiations and time
    // spent parsing urls.
    GlideUrl url = model;
    if (modelCache != null) {
      url = modelCache.get(model, 0, 0);
      if (url == null) {
        modelCache.put(model, 0, 0, model);
        url = model;
      }
    }
    int timeout = options.get(TIMEOUT);
    return new LoadData<>(url, new HttpUrlFetcher(url, timeout));
  }

  @Override
  public boolean handles(@NonNull GlideUrl model) {
    return true;
  }

  /**
   * The default factory for {@link HttpGlideUrlLoader}s.
   */
  public static class Factory implements ModelLoaderFactory<GlideUrl, InputStream> {
    private final ModelCache<GlideUrl, GlideUrl> modelCache = new ModelCache<>(500);

    @NonNull
    @Override
    public ModelLoader<GlideUrl, InputStream> build(MultiModelLoaderFactory multiFactory) {
      return new HttpGlideUrlLoader(modelCache);
    }

    @Override
    public void teardown() {
      // Do nothing.
    }
  }
}
```

我们回过来看ResourceCacheGenerator的startNext()  相关的说明在代码中进行注释：

```java
@Override
  public boolean startNext() {
      //这里不仅获取CacheKey，并且在getCacheKeys（）还将相关的loadData准备好了
    List<Key> sourceIds = helper.getCacheKeys();
    if (sourceIds.isEmpty()) {
      return false;
    }
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
      if (File.class.equals(helper.getTranscodeClass())) {
        return false;
      }
      throw new IllegalStateException(
         "Failed to find any load path from " + helper.getModelClass() + " to "
             + helper.getTranscodeClass());
    }
    while (modelLoaders == null || !hasNextModelLoader()) {
      resourceClassIndex++;
      if (resourceClassIndex >= resourceClasses.size()) {
        sourceIdIndex++;
        if (sourceIdIndex >= sourceIds.size()) {
          return false;
        }
        resourceClassIndex = 0;
      }

      Key sourceId = sourceIds.get(sourceIdIndex);
      Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
      Transformation<?> transformation = helper.getTransformation(resourceClass);
      // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
      // we only run until the first one succeeds, the loop runs for only a limited
      // number of iterations on the order of 10-20 in the worst case.
      currentKey =
          new ResourceCacheKey(// NOPMD AvoidInstantiatingObjectsInLoops
              helper.getArrayPool(),
              sourceId,
              helper.getSignature(),
              helper.getWidth(),
              helper.getHeight(),
              transformation,
              resourceClass,
              helper.getOptions());
        //获取缓存文件
      cacheFile = helper.getDiskCache().get(currentKey);
        //缓存文件存在，获取对应的modelLoaders，当modelLoaders不为空时退出循环进入下一步
      if (cacheFile != null) {
        sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData = modelLoader.buildLoadData(cacheFile,
          helper.getWidth(), helper.getHeight(), helper.getOptions());
        //查找到能够处理本次文件缓存的loadData
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }

    return started;
  }
```

同样的在DataCacheGenerator中的startNext也有类似的逻辑。我们知道在这里获取的是数据来源的缓存就好了。DataCacheGenerator和ResourceCacheGenerator都是从文件中获取图片，他们获取缓存文件的key有些不同。因为自己对这一块的理解还不够深入这里就不再多加描述了。

最后说一下SourceGenerator这里主要是从网络获取图片，具体的获取流程在前面已经描述了。同时它还会将从网络获取的图片缓存到本地。

这样glide的加载缓存流程就基本清楚了。

总结几个问题：

内存活动资源什么时候被加入、删除？

答：当从lru缓存中获取到或者网络加载成功的时候会将对应的资源加载活动资源缓存。整个活动资源是采用引用计数的方式来进行缓存，当他的计数器是0的时候会释放资源并将资源加入lru缓存

lru缓存的加入删除？

答：当活动资源被释放的时候会加入lru缓存。删除分为主动删除和被动删除：当内存资源不足的时候或者缓存满了的时候会主动删除最近使用较少的资源。如果是bitmap类型的话还会将之放进重用池进行处理。主动删除发生在当从lru缓存中获取并将资源加入活动缓存的时候。

关于两种文件缓存？他们两者之间的区别？因为个人理解不够暂时不做回答。