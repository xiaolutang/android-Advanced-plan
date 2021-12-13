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

