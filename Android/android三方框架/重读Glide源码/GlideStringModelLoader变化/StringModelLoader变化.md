

说起StringLoader如果没有自定义过Glide ModelLoader  可能并不知道这个是什么。在[Glide数据输入输出](https://juejin.cn/post/7036389645168410655) 我们提到过

> **ModelLoader:** 是一个泛型接口，最直观的翻译是模型加载器。ModelLoader标记了它能够加载什么类型的数据，以及加载后返回什么样的数据类型。

那么StringLoader就是能够将输入类型是String的数据进行加载。但是在[Glide数据输入输出](https://juejin.cn/post/7036389645168410655) 

我们定义 [VideoAssetUriLoader](https://juejin.cn/post/7036389645168410655#heading-12)加载视频asset目录下的首帧图的时候，输入一个String 也能够进行正确的加载这个到底是为什么，难道我们对于ModelLoader的认知是错误的吗？

# ModelLoader的获取过程

ModelLoader的获取会经历以下过程：

1. 从ModelLoaderRegistry获取指定输入类型的ModelLoader
2. 调用ModelLoader#handles判断能否真正处理当前model

这里就不粘贴源码：参考Registry#getModelLoaders，有兴趣可以自己看。

# StringLoader的实现

既然ModelLoader的获取是根据输入类型来决定的，那么我们输入字符串String的时候应该获取到的是一个StringLoader,我们猜测 它能加载我们的 [VideoAssetUriLoader](https://juejin.cn/post/7036389645168410655#heading-12)必然是在StringLoader实现了一些特别的逻辑处理。通过StringLoader代理我们的 [VideoAssetUriLoader](https://juejin.cn/post/7036389645168410655#heading-12)

通过查看代码StringLoader内部有三个实现了ModelLoaderFactory的内部类。他们都构建了一个输入类型为Uri的MoldelLoader

以AssetFileDescriptorFactory为例：

```java
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
```

可以发现通过在构造方法中传入输入类型是Uri的MoldeLoader,而AssetFileDescriptorFactory构建的输入，类型是Uri输出类型是AssetFileDescriptor,恰好与我们定义的 [VideoAssetUriLoader](https://juejin.cn/post/7036389645168410655#heading-12)输入输出相对应，因此通过输入String也能够加载asset目录下的视频资源。
