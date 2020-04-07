# 以终为始：拆解分析RecyclerView的设计与实现

通过这篇文章的阅读您可以了解

1. RecyclerView整体设计
2. RecyclerView的使用进阶
   1. 原生的灭霸效果
   2. 时间轴效果
   3. 鸿洋小飞机效果？

效果图：



RecyclerView是我们日常开发中非常重要的一个控件，小王所开发的项目非常多列表都是通过它来实现。小王平时关注的都是RecyclerView的使用,但是对于它的底层实现却知之甚少。为了加深对RecyclerView的理解，小王决定从RecyclerView的现有功能来反向分析RecyclerView的整体设计与实现。

# 如何自己实现一个RecyclerView?

想象这样的一个场景，如果你是一位Google工程师，需要你实现一个满足下面的特性RecyclerView提供给别的开发者使用，你会如何做呢？

1. 布局灵活多变，能够满足不同的布局特性
2. 能够显示大量的ui显示，不会因为布局过多导致崩溃

## 1.0版本

根据上面的特性，小王自定义了XwRecyclerView

我们知道，一个View的显示会经过三大流程：测量，摆放，绘制。而要提供灵活的布局我们可以通过策略模式针对不同的策略使用不同的布局方式。于是设计了LayoutManager，来负责子View的摆放

为了能够显示大量的ui我们需要对不在屏幕内的View进行回收，这样才不会因为加载的子View过多而导致oom。

因为这个RecyclerView是提供给别的开发者使用的，小王并不知道XwRecyclerViewd的子View会摆放那些。于是通过一个IViewProvider来为XwRecyclerView提供需要摆放的子View

这样XwRecyclerView的1.0版本的整体结构如下

![1586273418874](自己设计实现RecyclerView1.0.png)

