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

