Glide线程池配置

Glide如何实现优先级加载

线程池该如何配置



Glide用来进行图片加载，我们知道当页面暂停的时候，glide可以根据页面的生命周期，来暂停当前页面的请求，但是如果当前页面通过滑动加载大量图片，那么Glide是怎么进行图片加载的呢？是先调用的加载在前还是后调用的加载在前面呢？如果某个页面的部分图片需要优先被加载，那么Glide又该如何处理呢？

# Glide线程池的使用

在[Glide DecodeJob 的工作过程](https://juejin.cn/post/7035932291536797727)我们知道Glide在进行一次完成的数据加载会经历 ResourceCacheGenerator -->  DataCacheGenerator  -->  SourceGenerator 的三个过程变化。而在这个过程变化中会涉及到两个线程池的使用。

1. EngineJob#start开始本次请求

   ```java
   public synchronized void start(DecodeJob<R> decodeJob) {
     this.decodeJob = decodeJob;
       //如果是从 缓存中获取图片使用 diskCacheExecutor
     GlideExecutor executor = decodeJob.willDecodeFromCache()
         ? diskCacheExecutor
         : getActiveSourceExecutor();
     executor.execute(decodeJob);
   }
   
    private GlideExecutor getActiveSourceExecutor() {
        //如果useUnlimitedSourceGeneratorPool 为true 使用无限制的线程池
        //如果useAnimationPool 为true且如果useUnlimitedSourceGeneratorPool为false  使用动画线程池 否则使用sourceExecutor
       return useUnlimitedSourceGeneratorPool
           ? sourceUnlimitedExecutor : (useAnimationPool ? animationExecutor : sourceExecutor);
     }
   ```

   

2. EngineJob#reschedule重新进行调度

   ```java
   @Override
   public void reschedule(DecodeJob<?> job) {
     //此时线程池的使用逻辑和EngineJob#start不在文件中加载数据一致
     getActiveSourceExecutor().execute(job);
   }
   ```

# Glide线程池的配置

Glide Excutor参数初始化来自于GlideBuilder#build 而这些在不额外设置的情况下都来自于GlideExecutor。而GlideExecutor的所有线程池都是通过配置ThreadPoolExecutor来完成的。

## 初识ThreadPoolExecutor

ExecutorService是最初的线程池接口，ThreadPoolExecutor类是对线程池的具体实现，它通过构造方法来配置线程池的参数。

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

参数解释：

corePoolSize，线程池中核心线程的数量，默认情况下，即使核心线程没有任务在执行它也存在的，我们固定一定数量的核心线程且它一直存活这样就避免了一般情况下CPU创建和销毁线程带来的开销。我们如果将ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true，那么闲置的核心线程就会有超时策略，这个时间由keepAliveTime来设定，即keepAliveTime时间内如果核心线程没有回应则该线程就会被终止。allowCoreThreadTimeOut默认为false，核心线程没有超时时间。
maximumPoolSize，线程池中的最大线程数，当任务数量超过最大线程数时其它任务可能就会被阻塞。最大线程数=核心线程+非核心线程。非核心线程只有当核心线程不够用且线程池有空余时才会被创建，执行完任务后非核心线程会被销毁。
keepAliveTime，非核心线程的超时时长，当执行时间超过这个时间时，非核心线程就会被回收。当allowCoreThreadTimeOut设置为true时，此属性也作用在核心线程上。
unit，枚举时间单位，TimeUnit。
workQueue，线程池中的任务队列，我们提交给线程池的runnable会被存储在这个对象上。
线程池的分配遵循这样的规则：

当线程池中的核心线程数量未达到最大线程数时，启动一个核心线程去执行任务；
如果线程池中的核心线程数量达到最大线程数时，那么任务会被插入到任务队列中排队等待执行；
如果在上一步骤中任务队列已满但是线程池中线程数量未达到限定线程总数，那么启动一个非核心线程来处理任务；
如果上一步骤中线程数量达到了限定线程总量，那么线程池则拒绝执行该任务，且ThreadPoolExecutor会调用RejectedtionHandler的rejectedExecution方法来通知调用者。

## DiskCacheExecutor的配置过程

GlideExecutor提供了三个创建DiskCacheExecutor的方法，最终都会调用到有三个参数那个

```java
public static GlideExecutor newDiskCacheExecutor(
    int threadCount, String name, UncaughtThrowableStrategy uncaughtThrowableStrategy) {
  return new GlideExecutor(
      new ThreadPoolExecutor(
          threadCount /* corePoolSize */,
          threadCount /* maximumPoolSize */,
          0 /* keepAliveTime */,
          TimeUnit.MILLISECONDS,
          new PriorityBlockingQueue<Runnable>(),
          new DefaultThreadFactory(name, uncaughtThrowableStrategy, true)));
}
```

在默认创建的时候，调用的是无参数的那个，threadCount 值为1  即DiskCacheExecutor是一个核心线程数为1，没有非核心线程的线程池，所有任务在线程池中串行执行，Runnable的存储对象是PriorityBlockingQueue。

