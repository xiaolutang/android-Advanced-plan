# 浅析RecyclerView一 （ReCyclerView三大布局流程）

如何你没有阅读过RecyclerView的相关源码或者对于RecyclerView仅仅停留在简单的使用阶段，那么这篇文章可能会对你有所帮助。

RecyclerView可以说使我们日常开发中非常重要的一个ViewGroup。传闻中RecyclerView有五虎上将，他们分别是：

1. LayoutManager:负责RecyclerView的布局
2. Adapter  负责将数据转变成视图，使用的时候继承这个类。
3. ItemAnimation 子视图动画。
4. ItemDecoration 分割线。
5. Recycler 负责ViewHolder回收和创建。



文章借鉴了 

[郭神的抽丝剥茧心法修炼： 深剖RecyclerVie]: https://mp.weixin.qq.com/s/08LpubdLTUdYW10yAzomZg

来训练自己的源码阅读能力，

我给我自己设计了下面几个问题 ：

- RecyclerView的三大布局流程实现
- RecyclerView的adapter局部更新和 全局更新的区别

今天要分析的事RecyclerView的三大布局流程。他和传统的ViewGroup有什么不同？后续的问题将会在这一些列中的文章分析。



我们知道View呈现到界面上会经过下面几个步骤，如果不熟悉的话可以先进行学习再看下面的内容。

- measure 测量
- layout 布局
- draw 绘制

今天我们就通过源码来看看Recycler的实现

## **onMeasure**

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {//在没有设置LayoutManager时直接使用默认测量方式
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    //LinearLayoutManager和GridLayoutManager的isAutoMeasureEnabled返回是true
    //StaggeredGridLayoutManager根据不同的情况返回。
    //这里只分析返回值为true的情况。
    if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        final boolean measureSpecModeIsExactly =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        //如果RecyclerView的宽和高都是固定值（明确值或者match_parent）。那么 本次测量结束。
        //如果adapter未进行设置本次测量结束
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
        // consistency
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();

        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // if RecyclerView has non-exact width and height and if there is at least one child
        // which also has non-exact width & height, we have to re-measure.
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(
                    MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
    } else {
        ///省略代码
    }
}
```

这里我们暂时忽略 在onMeasure中调用的dispatchLayoutStep1和dispatchLayoutStep2方法。在onLayout中我们再来仔细分析他们。

## **onLayout**

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```

onLayout的逻辑非常简单，它几乎把所有的事情都交给了dispatchLayout方法。

```java
void dispatchLayout() {
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        // leave the state in START
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        // leave the state in START
        return;
    }
    mState.mIsMeasuring = false;
    //mState.mLayoutStep 在不考虑预取的情况下，只有dispatchLayoutStep1 和dispatchLayoutStep2会对他进行赋值。
    //我们假设给RecyclerView设定的事固定值。
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

接下来我们分析RecyclerView的layout三大步

**dispatchLayoutStep1：**

```java
/**
 * The first step of a layout where we;
 * - process adapter updates
 * - decide which animation should run
 * - save information about current views
 * - If necessary, run predictive layout and save its information
 */

//根据注释可以知道 这个方法 是进行View摆放的第一步，他会处理Adapter更新，决定需要运行那些动画 ，和保存当前View的一些信息。如果有必要 的话还会预取下一个layout数据。并保存它的信息。
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    processAdapterUpdatesAndSetAnimationFlags();
    //做Tv开发的朋友可能需要关注一下这个方法。他指到RecyclerView如何保存焦点信息。
    saveFocusInfo();
    //省略代码
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

**dispatchLayoutStep2**

这个方法主要是根据最终的状态实现VIew的layout摆放。并且这个方法可能被多次调用。在这里RecyclerView将子View的摆放交给了LayoutManager

```java
private void dispatchLayoutStep2() {
   
    // 省略代码
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);
    //省略代码
}
```

**dispatchLayoutStep3**

layout的最后一步，最要是恢复dispatchLayoutStep1保存的View信息，触发相应的动画和清楚一些信息。比如恢复焦点后清除关于焦点的信息。

## **draw**

Recycler的OnDraw方法仅仅绘制了分割线，它的子View的绘制在ViewGroup中实现，这里我们不进行分析。关于ViewGroup如何绘制子View可以自行查阅资料。

# LinearLayoutManager如何实现onLayoutChildren

# Adapter的局部与全局更新

adapter的数据更新都是通过其内部的mObservable对象注册的观察者来进行数据更新。RecyclerView的mObservable的具体实现类是：AdapterDataObservable

我们直接来分析一下他的notifyChanged和notifyItemRangeChanged方法。

## notifyChanged和notifyItemRangeChanged

```java
public void notifyChanged() {
    //RecyclerView对应的类为RecyclerViewDataObserver
    for (int i = mObservers.size() - 1; i >= 0; i--) {
        mObservers.get(i).onChanged();
    }
}

public void notifyItemRangeChanged(int positionStart, int itemCount,
                @Nullable Object payload) {
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onItemRangeChanged(positionStart, itemCount, payload);
            }
        }
```

我们看到RecyclerView在mObservers中注册的实现类RecyclerViewDataObserver

```java
@Override
public void onChanged() {
    assertNotInLayoutOrScroll(null);
    mState.mStructureChanged = true;

    //标记当前的屏幕上的View和 缓存View需要更新
    processDataSetCompletelyChanged(true);
    //如果没有等待更新的数据，直接请求重新测量布局绘制
    if (!mAdapterHelper.hasPendingUpdates()) {
        requestLayout();
    }
}

@Override
public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
    assertNotInLayoutOrScroll(null);
     //如果没有等待更新的数据，直接请求重新测量布局绘制
    if (mAdapterHelper.onItemRangeChanged(positionStart, itemCount, payload)) {
        triggerUpdateProcessor();//最后还是会调用到requestLayout请求重新布局
    }
}

/**
     * @return True if updates should be processed.
     */
    boolean onItemRangeChanged(int positionStart, int itemCount, Object payload) {
        if (itemCount < 1) {
            return false;
        }
        mPendingUpdates.add(obtainUpdateOp(UpdateOp.UPDATE, positionStart, itemCount, payload));
        mExistingUpdateTypes |= UpdateOp.UPDATE;
        return mPendingUpdates.size() == 1;
    }
```

通过上面的对比可以发现：

全部更新和局部更新的不同点是：

全部改变会标记当前页面和缓存数据（这个描述不够精确）全部需要更新。

notifyItemRangeChanged会通过mAdapterHelper添加一个UpdateOp对象。

相同点 ：

最后都会调用requestLayout请求重新布局。

通过上面的对比我们大胆的推测，notifyItemRangeChanged和notifyChanged对于View的改变是在RecyclerView的布局三大流程中。