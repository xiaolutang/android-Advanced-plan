# 前言

本篇文章不会太过专注于retrofit的使用，如果要了解它的使用细节，可以 参考其他的博客。本片文章大致思路如下，

1. retrofit 的使用
2. java 动态代理
3. retrofit的工作流程
4. 我所理解到的retrofit设计

# retrofit 的使用

引入retrofit依赖

```groovy
implementation 'com.squareup.retrofit2:retrofit:(insert latest version)'
```

声明接口

```java
public class WanAndroidApi {

    private static final String baseWanAndroidUrl = "https://www.wanandroid.com/";

    private static IWanAndroidAPI mIWanAndroidAPI;

    private static ConcurrentHashMap<String, List<Cookie>> cookieStore = new ConcurrentHashMap<>();
    public static IWanAndroidAPI getiWanAndroidAPI(){
        if(mIWanAndroidAPI == null){
            //为Retrofit提供自己的client
            OkHttpClient okHttpClient = new OkHttpClient.Builder()
                    .cookieJar(new CookieJar() {
                        @Override
                        public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
                            for (Cookie cookie : cookies)
                            {
                                System.out.println("cookies: " + cookie.toString());
                            }
                            cookieStore.put(baseWanAndroidUrl,cookies);
                        }

                        @Override
                        public List<Cookie> loadForRequest(HttpUrl url) {
                            List<Cookie> cookies = cookieStore.get(url.host());
                            return cookies != null ? cookies : new ArrayList<Cookie>();
                        }
                    })
                    .build();
            //初始化retrofit对象 
            Retrofit retrofit = new Retrofit.Builder()
                    .baseUrl(baseWanAndroidUrl)
                    .addConverterFactory(GsonConverterFactory.create())
                    .client(okHttpClient)
                    .build();
            mIWanAndroidAPI = retrofit.create(IWanAndroidAPI.class);
        }
        return mIWanAndroidAPI;
    }

    public interface IWanAndroidAPI{

        /**
         * Retrofit 基本使用
         * 获取wanAndroid首页banner
         * */
        @GET("banner/json")
        Call<JSONObject> getBanner();
    }
}
```

通过上面的步骤我们就可以使用IWanAndroidAPI 实例对象来获取Wandroid首页的轮播图了。

# Java动态代理

学习Retrofit和java动态代理有什么关系呢？因为动态代理是Retrofit中的一个比较关键的技术点。所以需要对他有一个比较好的理解，方面Retrofit的源码阅读。

Java的代理分为动态代理和静态代理。

> 所谓静态代理，其实质是自己手写(或者用工具生成)代理类，也就是在程序运行前就已经存在的编译好的代理类。但是，如果我们需要很多的代理，每一个都这么去创建实属浪费时间，而且会有大量的重复代码，此时我们就可以采用动态代理，动态代理可以在程序运行期间根据需要动态的创建代理类及其实例来完成具体的功能。

静态代理：

```java
public interface Subject {
    void visit();
}
```

```java
public class RealSubject implements Subject {
    @Override
    public void visit() {
        System.out.println( "RealSubject" );
    }
}
```

```java
public class ProxySubject implements Subject {
    RealSubject realSubject;

    public ProxySubject(RealSubject realSubject) {
        this.realSubject = realSubject;
    }

    @Override
    public void visit() {
        realSubject.visit();
    }
}
```

```java
public class Client {
    public static void main(String[] args){
        RealSubject realSubject = new RealSubject();
        ProxySubject proxySubject = new ProxySubject( realSubject );

        proxySubject.visit();
    }
}
```

动态代理

```java
public class DynamicProxy implements InvocationHandler {
    private Object object;

    public DynamicProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = method.invoke( object,args );
        return result;
    }
}
```

```java
public class Client {
    public static void main(String[] args){
        Subject realSubject = new RealSubject();
        DynamicProxy dynamicProxy = new DynamicProxy( realSubject );
        ClassLoader loader = realSubject.getClass().getClassLoader();
        Subject proxySubject = (Subject) Proxy.newProxyInstance( loader,new Class[]{Subject.class},dynamicProxy );
        proxySubject.visit();
    }
}
```

动态代理主要涉及到两个关键对象

Proxy：用于生产动态代理对象

InvocationHandler：用于处理被代理对象的相关 逻辑

**Retrofit中的动态代理：**

```java
 public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];
			//所有service 方法的调用都会走到这个位置
          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
```





参考：https://blog.csdn.net/justloveyou_/article/details/74203025