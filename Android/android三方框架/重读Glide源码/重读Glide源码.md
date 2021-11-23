为什么会有重读这个系列？原来分析过Glide加载流程，但是在较长的时候后，已经记不起Glide到底是怎么样来进行运转的了。这个说明一个问题我们并没有掌握到Glide的精髓。

# 如果你要开发一个图片加载框架

预设一个小问题？假设我们自己要设计一个图片加载框架，有哪些问题需要进行考虑

1. 内存缓存  使用lru将高频使用的图片缓存在内存
2. 文件缓存  没必要每次都从网络加载数据
3. 加载的时候根据显示大小进行采样，减少内存使用
4. 可定制的网络加载？可以自由定制使用什么方式来进行网络请求 比如okhttp  或者其他的网络加载框架 。这个是一个可选项
5. 图片加载生命周期管理，不可见页面的加载需要取消。
6. 图片加载优先级 最新的提交应该被优先加载。
7. bitmap重用池管理，不可能每次都重新分配对象
8. 加载源处理，不同的数据源怎么来进行解码  比如文件在用网络进行加载肯定不合适，视频加载封面图肯定和普通图片不一样
9. 大量图片加载的时候怎么确认对应的回调，像RecyclerView的图片加载怎么确认不出错。

接下来我们带着这些问题去看Glide源码，看看那些是我们没有想到的。



# Glide源码分析

因为分析过一次原来流程了，这次就不再跟着流程走了。而是围绕一些感兴趣的点来进行代码分析。

## Glide生命周期管理

Glide#with有多个重载方法，最终会变成两种类型，Activity和Fragment  Glide会在下面添加不可见的fragment然后利用fragment的生命周期来对请求的生命周期进行管理。

### 源码实现：

Glide#with内部会先调用getRetriever获取一个RequestManagerRetriever在由RequestManagerRetriever#get获取一个RequestManager，RequestManagerRetriever的主要工作职责是从activity或fragment获取对应的Requestmanager。RequestManagerRetriever有多个get重载

![image-20211123090528893](\RequestManagerRetriever#get重载.png)

以传递Fragment为例：RequestManager的获取分为两个流程

1. 查找对应的SupportRequestManagerFragment
2. 构建RequestManager

**查找：**

```java
@NonNull
  public RequestManager get(@NonNull Fragment fragment) {
//    ...
    if (Util.isOnBackgroundThread()) {
      return get(fragment.getActivity().getApplicationContext());
    } else {
      FragmentManager fm = fragment.getChildFragmentManager();
      return supportFragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
    }
  }

@NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
      //获取一个fragment，如果没有的话会创建一个SupportRequestManagerFragment
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
        //factory可以通过注解指定，没有的话就是用默认的factory
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```

**构建RequestManager**

```java
private static final RequestManagerFactory DEFAULT_FACTORY = new RequestManagerFactory() {
    @NonNull
    @Override
    public RequestManager build(@NonNull Glide glide, @NonNull Lifecycle lifecycle,
        @NonNull RequestManagerTreeNode requestManagerTreeNode, @NonNull Context context) {
      return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
    }
  };

```

RequestManager构造方法内部通过

```
lifecycle.addListener(this);
```

实现了对SupportRequestManagerFragment的生命周期的监听。从而实现对当前页面（Activity/Fragment的加载生命周期管理）

### 小结：

1. Glide#with方法传递参数的优先级是  fragment > activity >  View > application   其中activity和view应该是比较有争议的。如果View直接在activity中，肯定是直接传递activity好，如果view在fragment  glide内部会遍历所有fragment，然后查找到所对应的fragment。这个过程会经历多次循环，相对比较耗时。
2. Requestmanager通过监听SupportRequestManagerFragment的生命周期实现对图片加载生命周期进行管理。

## Glide输入、输出变化



