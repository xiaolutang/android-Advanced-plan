问题：

同一个分发器，同一个域名最多5个请求？会不会限制网络请求的数量？但是同一个app中大多数网络请求都是针对同一个域名吧？

同一个分发器最多同时与64个线程同时运行？会不会因为一些请求卡主导致后续的请求起不来？

都在说责任链，责任链式怎么构成运转的？和View的事件分发责任链有什么区别？

addInterceptor和addNetworkInterceptor的区别。

网络请求的连接池是什么？okhttp又是怎么来实现的？



行文结构：

请求流程

责任链的运转原理。

作为一个Android开发者，okhttp是我们常用的一个网络开发框架。大多数人都知道okhttp好，但是好在什么地方却难以诉说。今天我们就来探究okhttp设计理念和实现思路都有哪些值得我们学习。

为了防止深陷源码细节，我们提前预设几个问题

1. 同一个分发器，同一个域名最多5个请求？会不会限制网络请求的数量？但是同一个app中大多数网络请求都是针对同一个域名吧？
2. 同一个分发器最多同时与64个线程同时运行？会不会因为一些请求卡主导致后续的请求起不来？
3. 都在说责任链，责任链式怎么构成运转的？和View的事件分发责任链有什么区别？
4. addInterceptor和addNetworkInterceptor的区别。
5. 网络请求的连接池是什么？okhttp又是怎么来实现的？
6. okhttp自带的各个拦截器的作用

# OkHttp请求流程

okhttp发起请求需要三步：

1. 创建OkHttpClient对象
2. 创建Request对象
3. OkHttpClient链接Request生成RealCall并通过RealCall发起网络请求

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .build();
Request request = new Request.Builder()
        .url(url)
        .build();
//异步请求
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
//同步请求
Response response = okHttpClient.newCall(request).execute();
```

OkhttpClient和Request独立变化，RealCall通过桥接的方式将OkhttpClient和Request关联。

## 同步执行过程：

同步通过调用ReallCall#execute()执行

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    timeout.enter();
    eventListener.callStart(this);
    try {
      //通过调度器将当前的call加入同步执行队列
      client.dispatcher().executed(this);
      //从拦截器链获取请求的响应  这里先不解释这个，后面详细说明
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      e = timeoutExit(e);
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

## OkHttp Dispatcher 调度器

Dispatcher 负责对所有OkHttp网络请求进行维护。它的内部维护了三个双端队列

```java

  /** Ready async calls in the order they'll be run. */
//已经准备好可以运行的异步请求
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
//正在运行的异步请求
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
//正在运行的同步请求
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

对于同步请求则直接加入runningSyncCalls 队列，对于异步请求先将请求加入readyAsyncCalls表示当前请求已经准备好可以运行。在条件允许的时候从readyAsyncCalls中移除，加入runningAsyncCalls。这个条件参考后续异步执行的过程。

同时异步任务通过Dispatcher#executorService配置的线程池来执行

```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

这个线程池每接收一个任务就会开启一个线程执行。更多关于线程池配置的东西可以参考我原来的一篇文章 

[Android中的线程池](https://juejin.cn/post/7047166823380287518)

## 异步执行过程：

同步通过调用ReallCall#enqueue()执行

```java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    //构建一个AsyncCall 通过Dispatcher enqueue 放入队列
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

AsyncCall是RealCall的一个内部类，它继承了抽象类NamedRunnable。

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```

NamedRunnable实现了Runnable接口，当它被线程池执行的时候会调用它的execute方法。

### Dispatcher#enqueue

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
      //将异步call放入异步队列 这个队列中的数据表示已经准备好可以执行
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
  }
```

### promoteAndExecute()函数

```java
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      //从readyAsyncCalls 取出可以执行的任务。
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();
		//如果当前正在运行的异步网络请求数量大于  maxRequests maxRequests的默认值是64 退出
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        //如果 如果当前正在运行的线程池当中同一个域名的连接个数大于  
        // maxRequestsPerHost（默认值是5）跳过当前继续下一次循环
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        //收集可以执行的异步任务
        executableCalls.add(asyncCall);
        //将任务加载到正在执行的队列
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      //通过线程池执行
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

从promoteAndExecute 可以看出，虽然OkHttp的线程池配置是可以创建无线的线程执行任务，但是它通过Dispatcher对线程池的入口进行管理。有效的控制了最大的并发线程数，并且在没有网络请求的时候因为没有核心线程，所有非核心线程超时后会被回收，不会占用过多的系统资源。

### AsyncCall#execute

AsyncCall 因为继承 NamedRunnable 在调用AsyncCall #executeOn是会将自身传递给线程池，线程池在执行的时候会调用它的execute

```java
void executeOn(ExecutorService executorService) {
    	...省略代码
       executorService.execute(this);
    ...省略代码
    }

@Override protected void execute() {
      boolean signalledCallback = false;
      timeout.enter();
      try {
        Response response = getResponseWithInterceptorChain();
         ...省略代码
    }
  }
```

## getResponseWithInterceptorChain的执行过程

能够看到的，不论是同步请求还是异步请求最后都会执行getResponseWithInterceptorChain()来获得请求的Response。

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    //构建拦截器
    List<Interceptor> interceptors = new ArrayList<>();
    //优先处理的是OkHttpClient设置的拦截器
    interceptors.addAll(client.interceptors());
    //重定向拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      //添加OkHttpClient设置的网络拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //构建链路
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
	//链路开始执行
    return chain.proceed(originalRequest);
  }
```

## RealInterceptorChain#proceed的运转过程

OkHttp经典的责任链处理逻辑开始于RealInterceptorChain#proceed。

```java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
     ...省略代码

    // Call the next interceptor in the chain.
    //构建下一个请求节点。index的初始值为0，人次创建RealInterceptorChain时 index + 1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    //根据构建RealInterceptorChain 时传入的index获取对应的拦截器
    Interceptor interceptor = interceptors.get(index);
    // 
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

### RetryAndFollowUpInterceptor的工作过程

