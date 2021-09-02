# ANR学习笔记

# 前言

对于开发者而言，我们肯定碰到过anr 全称 Applicatipon No Response  。那么我们遇到anr时该怎么处理呢？本文将从下面的几个方面探索学习anr 的监控与解决。

- ANR的设计目的
- ANR发生场景
- ANR相关日志分析
- ANR监控

# ANR设计目的

定义引用于

[今日头条 ANR 优化实践系列 - 设计原理及影响因素]: https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&amp;mid=2247488116&amp;idx=1&amp;sn=fdf80fa52c57a3360ad1999da2a9656b&amp;chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&amp;token=569762407&amp;lang=zh_CN#rd

> ANR 全称 Applicatipon No Response；Android 设计 ANR 的用意，是系统通过与之交互的组件(Activity，Service，Receiver，Provider)以及用户交互(InputEvent)进行超时监控，以判断应用进程(主线程)是否存在卡死或响应过慢的问题，通俗来说就是很多系统中看门狗(watchdog)的设计思想。

# 发生anr的场景

一般而言主线程有耗时操作会导致卡顿，卡顿超过系统设置的阈值，就会触发anr。而Android 主线程任务的执行是通过handler发送msg到消息队列，然后通过looper取出msg来进行处理。

```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        ...

        for (;;) {
        //从消息队列取出消息
            Message msg = queue.next(); // might block
            ...
           //处理消息
                msg.target.dispatchMessage(msg);
            ...
    }
```

说明造成卡顿的有两个位置

1. queue.next()阻塞
2. dispatchMessage()耗时过久

那么是不是只要这两个方法执行不超过阈值，就不会发生ANR呢？

在 [今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488116&idx=1&sn=fdf80fa52c57a3360ad1999da2a9656b&chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&token=569762407&lang=zh_CN#rd)   中以广播为例介绍了anr的监测过程，我们这里用service来进行举例说明



activity生命周期方法会发生anr吗？

界面卡顿和anr的关系。

# anr分析

1. 看trace(trace.txt)  文件
2. 看anr信息

anr原理

anr 卡顿监控





参考：

[今日头条 ANR 优化实践系列 - 监控工具与分析思路]: https://juejin.cn/post/6942665216781975582
[今日头条 ANR 优化实践系列 - 设计原理及影响因素]: https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&amp;mid=2247488116&amp;idx=1&amp;sn=fdf80fa52c57a3360ad1999da2a9656b&amp;chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&amp;token=569762407&amp;lang=zh_CN#rd
[卡顿、ANR、死锁，线上如何监控？]: https://juejin.cn/post/6973564044351373326
[干货：ANR日志分析全面解析]: https://juejin.cn/post/6971327652468621326

