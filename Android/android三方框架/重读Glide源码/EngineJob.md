ResourceCacheGenerator的工作过程

startNext方法

```java
public boolean startNext() {
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
      cacheFile = helper.getDiskCache().get(currentKey);
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
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }

    return started;
  }
```





```java
@NonNull
  public <Model, TResource, Transcode> List<Class<?>> getRegisteredResourceClasses(
      @NonNull Class<Model> modelClass,
      @NonNull Class<TResource> resourceClass,
      @NonNull Class<Transcode> transcodeClass) {
    List<Class<?>> result =
        modelToResourceClassCache.get(modelClass, resourceClass, transcodeClass);

    if (result == null) {
      result = new ArrayList<>();
      //获取能够加载当前model加载出的数据class
      List<Class<?>> dataClasses = modelLoaderRegistry.getDataClasses(modelClass);
      for (Class<?> dataClass : dataClasses) {
        List<? extends Class<?>> registeredResourceClasses =
            decoderRegistry.getResourceClasses(dataClass, resourceClass);
        for (Class<?> registeredResourceClass : registeredResourceClasses) {
          List<Class<Transcode>> registeredTranscodeClasses = transcoderRegistry
              .getTranscodeClasses(registeredResourceClass, transcodeClass);
          if (!registeredTranscodeClasses.isEmpty() && !result.contains(registeredResourceClass)) {
            result.add(registeredResourceClass);
          }
        }
      }
      modelToResourceClassCache.put(
          modelClass, resourceClass, transcodeClass, Collections.unmodifiableList(result));
    }

    return result;
  }
```

