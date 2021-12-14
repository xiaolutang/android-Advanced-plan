Glide是我们日常开发中的常用的图片加载框架，也是面试中常问的问题。

用尽可能简单的方式把Glide原理讲清楚。



# 为什么使用Glide?

面试官：看你简历上说熟练使用Glide，能够说一说为什么项目上图片加载框架使用的是Glide而不是其它呢？

> 我进入项目小组就是使用的Glide图片加载框架，它能够满足我们日常的业务需求，所以就一直使用的Glide。

如果你是像上面那样回答的面试官，我么我猜测大概率你在项目中是一个执行者，作为辅助工程师。

1. 使用方便,API简洁。with、load、into 三步就可以加载图片
2. 生命周期自动绑定，根据绑定的Activity或Fragment生命周期管理图片请求
3. 支持多级配置：应用、单独页面（Activity/Fragment）、单个请求进行独立配置。
4. 高效缓存策略，两级内存 ，两级文件。
5. 支持多种图片格式(Gif、WebP、Video), 扩展灵活



# Glide加载流程

Glide的加载过程大致如下，Glide#with获取与生命周期绑定的RequestManager，RequestManager通过load获取对应的RequestBuilder。根据RequestBuilder构建对应的Request,Target 将Request,Target 交给RequestManager进行统一管理。调用RequestManager#track开始进行图片请求。request通过Engine分别尝试从活动缓存、Lru缓存、文件缓存中加载图片，当以上的缓存中都不存在对应的图片后，会从网络中获取。而网络获取大致可以分成，ModelLoader模型匹配，DataFetcher数据获取，然后经历解码、图片变换、转换。如果能够进行缓存原始数据，还会将解码的数据进行编码缓存到文件。



# Glide生命周期管理

Glide通过给Fragment/Activity插入一个不可见的Fragment,通过监听该Fragment的生命周期。来实现对应的请求管理。但是需要注意的是，如果在Fragment中使用Activity，在图片的请求过程中Fragment被销毁，但是请求并没有结束，会造成内存泄漏。

# Glide RequestOptions请求参数配置

GlideReuqestOptions参数配置分为三级，第一级作用于Glide ,在RequestManger的构造方法通过GlideContext获取 进行配置处理。第二即作用于RequestManger如果我们要针对某个

RequestManger进行配置处理。那么需要使用RequestManger#applyDefaultRequestOptions来对默认的配置进行更新。第三级作用于RequestBuilder，通过它的apply方法为每一个请求的配置进行更新。

# Glide缓存管理

Glide缓存分为，内存缓存，文件缓存，网络缓存。其中内存缓存又分为活动资源缓存，Lru缓存，文件缓存分为资源缓存和原始数据缓存。

**内存缓存**

活动资源缓存将每一个正在使用的图片加入活动资源缓存，每增加一个使用对象引用计数器加一，否则减一。

Lru缓存是按照最近最少使用的原则来对图片内存缓存进行维护，当Lru缓存满了的时候，优先移除访问时间最久的那个。

活动资源缓存以较小的代价，维护当前在内存中使用的图片资源，减轻Lru缓存的压力，提高缓存效率。

**文件缓存**

资源文件缓存是根据当前所需要的资源类型，图片大小等特定信息进行的缓存，当从资源缓存中获取数据的时候，不需要进行解码操作，获取的数据可以直接进行使用。

原始数据是根据网络加载的数据，直接进行缓存。使用的时候还需要重新进行解码，转换的流程。

# 内存管理

Glide的内存管理有两块，一、OOM的防治；二、内存抖动。

## OOM

图片加载非常重要的一点就是OOM的防治,Glide通过图片采样，弱引用、生命周期绑定等方式，减小加载到内存的的图片大小，及时清除不需要在使用对象的引用，从而减小OOM的概率。

### 图片的加载

Glide针对较大的图片，会根据当前ui的显示大小与实际大小的比例，进行采样计算从而减小图片在内存中的占用。一般而言

图片的大小  =  图片宽 X  图片高 X 每个像素占用的字节数。

对于资源文件夹下的图片：

图片的高 = 原图高 X (设备的 dpi / 目录对应的 dpi )

图片的宽 = 原图宽 X (设备的 dpi / 目录对应的 dpi )

### onlowMemory/onTrimMemory

Glide通过实现ComponentCallbacks2并将其注册进Applition,  当内存过低的时候会调用onlowMemory，在onlowMemory 中Glide会将一些缓存的内存进行清除，方便进行内存回收，当onTrimMemory被调用的时候，如果level是系统资源紧张，Glide会将Lru缓存和BitMap重用池相关的内容进行回收。如果是其他的原因调用onTrimMemory，Glide会将缓存的内容减小到配置缓存最大内容的1/2。

### 借助弱引用

Glide通过RequestManager管理图片请求，而RequestManager内部是通过RequestTracker和TargetTracker来完成的。他们持有的方式都是弱引用。

## 内存抖动的处理.

Glide通过重用池技术，将一些常用的对应进行池话，比如图片加载相关的EngineJob DecodeJob等一下需要大量重复使用创建的对象，通过对象重用池进行对象重用。

BitmapPool对Bitmap进行对象重用。在对图片进行解码的的时候通过设置BitmapFactory.Options#inBitmap来达到内存重用的目的。

> 在 `Android 3.0（API 级别 11）`开始，系统引入了 `BitmapFactory.Options.inBitmap` 字段。如果设置了此选项，那么采用 `Options` 对象的解码方法会在生成目标 `Bitmap` 时尝试复用 `inBitmap`，这意味着 `inBitmap` 的内存得到了重复使用，从而提高了性能，同时移除了内存分配和取消分配。不过 `inBitmap` 的使用方式存在某些限制，在 `Android 4.4（API 级别 19）`之前系统仅支持复用大小相同的位图，4.4 之后只要 `inBitmap` 的大小比目标 `Bitmap` 大即可



# 列表页图片加载数据错乱

由于RecyclerView、ListView的View复用机制，可能出现第一个item的图片显示在第10个上，这个明显是错误的。Glide通过给Target#setRequest，将Target与Request关联。针对View类型的Target，setRequest的实质是给View设置tag，通过tag保存request,当下一个持有相同View的Target到来的时候，也可以取出原来的request,并将其取消。但是针对非View类型的target,如果要使用这个特性，我们需要提供原来的在使用的target，而不是像View一样重新创建一个新的对象。

# Glide中的线程&线程池

关于Glide中的线程线程池，准备说两个方面

- 图片加载回调
- Glide的线程池配置

## 图片加载回调

Glide有两种图片加载方式into和submit  通过into加载的图片会通过Executors#MAIN_THREAD_EXECUTOR回调到主线程。

而通过submit的进行回调的会通过Executors#DIRECT_EXECUTOR在当前线程进行处理。

## Glide线程池配置

线程作为cpu调度的最小单元，每一次的创建和回收都会有较大的消耗，通过使用线程池可以

- 降低资源消耗：通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- 提高响应速度：当任务到达时，可以不需要等待线程创建就能立即执行。
- 提高线程的可管理性：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，监控和调优。
- 有效的控制并发数

Glide中提供了四种线程池配置。

- DiskCacheExecutor  该线程池只有一个核心线程，没有非核心线程，所有任务在线程池中串行执行。在Glide中常用与从文件中加载图片。
- SourceExecutor  该线程也只有核心线程没有非核心线程，与DiskCacheExecutor 的不同之处在于核心线程的数量根据CPU的核数来决定。如果cpu核心数超过4则核心线程数为4  如果Cpu核心数小于4那么使用Cpu核心数作为核心线程数量。在Glide中长用来从网络中加载图片。
- UnlimitedSourceExecutor 没有核心线程，非核心线程数量无限大。这种类型的线程池常用于执行量大而快速结束的任务。在所有任务结束。在所有任务结束后几乎不消耗资源。
- AnimationExecutor  没有核心线程，非核心线程数量根据Cpu核心数来决定，当Cpu核心数大于等4时 非核心线程数为2，否则为1。

# Glide如何加载不同类型的资源

Glide通过RequestManager#as方法确定当前请求Target最终需要的资源类型。通过load方法确定需要加载的model资源类型，资源的加载过程经历ModelLoader的model加载匹配，解码器解码，转码器的转换，这几个过程构建成一个LoadPath 。而每一个LoadPath 又包含很多的DecodePath，DecodePath的主要作用是将ModelLoader加载出来的数据进行解码，转换。

Glide会遍历所有可能解析出对应数据的LoadPath 直到数据正真解析成功。

