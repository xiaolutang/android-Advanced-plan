在上一篇学习笔记

[Glide加载流程分析]: https://juejin.im/post/5eaad7e1f265da7bd802a819

中我们大致了解了Glide的加载流程，这里我们主要分析一下Glide的缓存。

Glide的缓存可以说考虑的非常全面分为下面的几个方面

- 内存缓存
  - 活动资源
  - Lru内存缓存
- 磁盘缓存
- 网络

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