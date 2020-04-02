# 前言

从进入公司项目上就在使用Retrofit,一直搞不明白retrofit是如何进行网络加载的除了问题也经常不知道怎么进行查找，正好最近研究了一下Retrofit的实现。记录一篇我的学习记录希望对大家了解Retrofit有所帮助。

# retrofit入门使用

retrofit的基本使用推荐直接学习

[http://square.github.io/retrofit/官方教程]: 



或者其他的一些博客：

https://www.jianshu.com/p/308f3c54abdd

https://blog.csdn.net/shusheng0007/article/details/81335264

# retorfit实现原理

## 动态代理返回Call对象

创建接口对象的方法是Retrofit的create()方法其代码如下：

```java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          //核心就是下面这两句代码
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

可以看到这里使用了java的动态代理返回了接口实例对象。在invoke方法内最后几句代码特别重要，我理解为这个是retorfit实现的核心。

# retorfit的简单实现

