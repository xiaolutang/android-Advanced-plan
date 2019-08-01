今天我们要学习的内容是《Android开发艺术探索》第三章View的事件体系。作者大致从五个方面进行知识的讲解。

# 基础知识的准备

作者为了让我们更加容易理解后续的知识，需要我们掌握几个和View相关的基础知识。

1. 什么是View?

   View是Android所有控件的基类，是一种界面的抽象。Android中的View可以分成两类，View和ViewGroup。整个界面通过ViewGroup包裹View和ViewGroup形成界面树。

2. View的位置参数

   View的位置主要由它的四个顶点来决定，分别对应top，left,right,bottom。这些坐标都是相对父容器来说的，top和bottom对应上边界，left和right对应左边界。效果图如下：

   ![View的坐标](./View的坐标.png)

   从3.0开始View新加了几个额外的参数。x、y、translationX、translationY。他们和left的关系如下

   x = left + translationX

   y = top + translationY

   可以这样来理解他们的含义。

   translationX是View在水平放下上的平移距离。X是View在视觉上看起来相对父容器的位置。

   同样的在竖直方向上translationY是竖直方向上的移动距离，y是View在视觉上相对父容器的位置。

3. MotionEvent

   手指接触屏幕产生的一些列事件的封装，通过MotionEvent这个对象我们可以得到当前事件的X/Y坐标。系统提供了两种方法：getX/getY和getRawX/getRawY 他们的区别是getX/getY返回的是相对当前View左上角的X和Y坐标，而getRawX/getRawY返回的是相对手机屏幕左上角的X和Y坐标

4. TouchSlop

   系统所能识别的被认为是滑动的最小距离，这个值和设备有关，不同的设备可能值不同。

5. VelocityTracker

   速度追踪，用于计算滑动过程中的速度。

   使用的时候先计算在获取，不使用的时候需要调用clear来重置并回收内存。

6. GestureDetector

   手势识别

7. Scroller

   用于实现View弹性滑动

# 滑动&弹性滑动的实现

在Android滑动几乎是应用的标配，几乎所有的App都使用到了滑动。滑动的实现常见的有三种方式

## 使用ScrollTo/ScrollBy实现滑动

可以看看scrollTo和ScrollBy的源码实现

```java
	/**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
    
    **
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }
```

可以看到ScrollBy最终调用的是scrollTo。ScrollBy实现相对位置滑动，scrollTo实现绝对位置滑动。View的滑动关键参数mScrollX和mScrollY可以通过getScrollX/getScrollY获取。mScrollX的值总是等于View左边缘和View内容左边在水平方向上的距离，mScrollY的值总是等于View上边缘和View内容上边在竖直方向上的距离。

从左向右滑mScrollX为负，反之为正。

从上往下滑mScrollY为负，反之为正。

记住 mScrollX/mScrollY = View的边界 - View内容的边界

ScrollTo/ScrollBy实现的是View内容的滑动。



## 使用动画

通过改变translationX/translationY来实现View的移动。移动的是View

## 改变布局参数

通过改变LayoutParams的margin来实现移动的效果。加入View右移100px我们需要将LayoutParams的marginLeft添加100px

## 三种滑动方式对比

ScrollTo/ScrollBy滑动的是View内容，使用简单。

动画实现复杂的动画效果，使用简单。

改变布局参数，操作复杂。



## 借助Scroller实现弹性滑动

典型使用方法：

```java
private Scroller mScroller = new Scroller(context)

//缓慢滚动到指定位置
private void smoothScrollTo(int destX, int destY){
	int scrollX = getScrollX();
	int deltax = destX - scrollX;
	mScroller.startScroll(scrollX,0,deltax,0,1000);
	invalidate()
}

@Override
public void computeScroll(){
	if(mScroller.computeScrollOffset()){
		scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
		postInvalidate();
	}
}
```

Scroller的弹性滑动实现原理：

Scroller本身自己不能实现滑动，在调用startScroll()后需要更新View的界面（比如调用 invalidate），而更新界面会调用View的draw方法，draw方法会调用computeScroll。而computeScroll又会调用postInvalidate。这样一次一次反复调用最后实现滑动玩成。Scroller在滑动的过程中的功能是根据时间获取当前应该滑动的位置。

## 借助动画实现弹性滑动

动画本身就是一种渐进过程，可以非常方便的实现弹性滑动。其本质是在动画的改变过程中动态改变相关的属性。

比如translationX/Y 或者mScrollX/Y

## 使用延时策略实现弹性滑动

通过handler延时分多次滑动目标距离，比如1000ms中滑动100px，那么每100ms滑动10px知道滑动距离为100px

# View的事件分发机制

在Android中View的事件分发主要由几个方法完成：

dispatchTouchEvent() 事件分发。

onInterceptTouchEvent() 事件拦截。注意这个只有ViewGroup有这个方法。

onTouchEvent() 事件处理。

当事件传递到ViewGroup的disoatchTouchEvent时候,它会先判断自身是否需要拦截处理，如果需要进行拦截处理就自己处理，如果不拦截则交给它的子View（包含ViewGroup和View）处理，如果它的子View不处理，则它自身的onTouchEvent会被调用。

![View的事件分发机制](./View的事件分发机制.png)

如上所示View的事件处理流程大概就是这样。

### 从源码中理解事件分发机制

ViewGroup 部分dispatchTouchEvent()代码

```java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
          if (!disallowIntercept) {
             intercepted = onInterceptTouchEvent(ev);
             ev.setAction(action); // restore action in case it was changed
           } else {
             intercepted = false;
           }
} else {
     // There are no touch targets and this action is not an initial down
     // so this view group continues to intercept touches.
     intercepted = true;
}
```

从这里我们可以看出ViewGroup在两种情况下会判断是否需要拦截当前事件，事件类型为ACTION_DOWN或mFirstTouchTarget不为空。ACTION_DOWN是指手指按下屏幕的那一瞬间，点击事件的开始。当ViewGroup的子元素处理成功时mFirstTouchTarget会被赋值并指向子元素。一旦ViewGroup拦截本次事件，mFirstTouchTarget会被置空，当ACTION_MOVE和ACTION_UP到来的时候。就不会再对是否拦截进行判断，并且同一系列事件的其他事件都会交给它处理。

从代码中我们也可以看到ViewGroup对事件的拦截还受mGroupFlags影响。如果mGroupFlags包含FLAG_DISALLOW_INTERCEPT那么ViewGroup不会对本次事件进行拦截。需要注意的是在ACTION_DOWN来临的时候ViewGroup会调用resetTouchState() 重置这个标志。

```java
private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        mNestedScrollAxes = SCROLL_AXIS_NONE;
    }
```

即一旦这个标志位被设置那么ViewGroup将无法拦截ACTION_DOWN以外的事件。

当ViewGroup不拦截事件的时候，事件会向下分发交给它的子View进行处理。在进行分发的过程中会调用到dispatchTransformedTouchEvent（）来进行处理。

```java
/**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) 
```

这里没有将代码全部贴出来，但是从方法的注释中我们可以简单的理解为：当参数child为空那么会调用ViewGroup的super.dispatchTouchEvent()即自己处理本次事件，否则调用child.dispatchTouchEvent()将事件传递给子View进行分发处理。如果ViewGroup拦截本次事件或者没有找到合适的子View处理本次事件，mFirstTouchTarget就会为 null，在本次事件序列不会再调用onInterceptTouchEvent（）方法。

View对事件的处理：

View的dispatchTouchEvent()部分代码

```
ListenerInfo li = mListenerInfo;
if (li != null && li.mOnTouchListener != null
       && (mViewFlags & ENABLED_MASK) == ENABLED
       && li.mOnTouchListener.onTouch(this, event)) {
           result = true;
}

 if (!result && onTouchEvent(event)) {
        result = true;
}
```

从这个代码中我们可以看出，OnTouchListener 优先于onTouchEvent(),如果设置了 OnTouchListener 并且返回true那么OnTouchEvent（）方法不会被调用。

# View的滑动冲突

常见的滑动冲突主要分成三种场景：

![滑动冲突场景](./滑动冲突场景.png)

对于滑动冲突的处理规则：

对于场景1：我们可以根据滑动方向决定哪个View处理本次事件交给哪个View处理

对于场景2：我们可以根据自身的业务逻辑来决定哪个View进行滑动

对于场景3：其本质是场景1和场景2的嵌套，我们只需要分别对他们进行处理即可。

### 简单理解ViewPager对于滑动冲突的处理

我们在ViewPager中放置一个RecyclerView 不论它是水平还是竖直滚动，ViewPager总是先将滑动事件交给它的子View进行处理，在子View滑动玩成后再滑动自身。这到底是为什么呢？

ViewPager没有重写了onInterceptTouchEvent（）和onTouchEvent（）由前面的View的事件分发机制我们知道，一旦ViewGroup的onTouchEcent()被调用就意味着它自己处理这一系列事件，即mFirstTouchTarget != null为false。子View不会再接收到这一系列事件。即ViewPager对于事件的处理在onInterceptTouchEvent()中。

ViewPager对于滑动事件的拦截处理部分关键代码

```java
if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                        && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    // Nested view has scrollable area under this point. Let it be handled there.
                    mLastMotionX = x;
                    mLastMotionY = y;
                    mIsUnableToDrag = true;
                    return false;
                }
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {
                    // The finger has moved enough in the vertical
                    // direction to be counted as a drag...  abort
                    // any attempt to drag horizontally, to work correctly
                    // with children that have scrolling containers.
                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");
                    mIsUnableToDrag = true;
                }
```

总结有两个个点：

1. 调用canScroll（）判断子ViewGroup是否可以进行水平滑动，如果可以的话，交给子ViewGroup处理。
2. 判断滑动的方向，如果Y方向的滑动距离大于X方向上的滑动距离。则不进行拦截，反之进行拦截。

这就是为什么ViewPager在水平方向上子ViewGroup总是优先于ViewPager进行滑动。竖直方向上ViewPager直接将滑动事件交给子ViewGroup处理。

对于第二条我们都很好理解，但是对于第一条这个是怎么做到的呢？我们直接来看看canScroll（）源码。

```java
/**
     * Tests scrollability within child views of v given a delta of dx.
     *
     * @param v View to test for horizontal scrollability
     * @param checkV Whether the view v passed should itself be checked for scrollability (true),
     *               or just its children (false).
     * @param dx Delta scrolled in pixels
     * @param x X coordinate of the active touch point
     * @param y Y coordinate of the active touch point
     * @return true if child views of v can be scrolled by delta of dx.
     */
    protected boolean canScroll(View v, boolean checkV, int dx, int x, int y) {
        if (v instanceof ViewGroup) {
            final ViewGroup group = (ViewGroup) v;
            final int scrollX = v.getScrollX();
            final int scrollY = v.getScrollY();
            final int count = group.getChildCount();
            // Count backwards - let topmost views consume scroll distance first.
            for (int i = count - 1; i >= 0; i--) {
                // TODO: Add versioned support here for transformed views.
                // This will not work for transformed views in Honeycomb+
                final View child = group.getChildAt(i);
                if (x + scrollX >= child.getLeft() && x + scrollX < child.getRight()
                        && y + scrollY >= child.getTop() && y + scrollY < child.getBottom()
                        && canScroll(child, true, dx, x + scrollX - child.getLeft(),
                                y + scrollY - child.getTop())) {
                    return true;
                }
            }
        }

        return checkV && v.canScrollHorizontally(-dx);
    }
```

可以看到它会循环查看ViewPager中的所有子ViewGroup及其子ViewGroup下面的所有ViewGroup是否能够处理这次水平滑动。这个东西理解起来有点绕。简单的来说就是对于一个ViewGroup而言只要它的向上递归查找父容器能够找到ViewPager,那么它的水平滑动就会优先于ViewPager处理。如下图RecyclerView可以理解成能够水平滑动的ViewGroup

![ViewPager对滑动事件的处理](./ViewPager对滑动事件的处理.png)

滑动冲突的处理：

1. 外部拦截法：指的是点击事件都经过父容器进行处理，如果父容器需要就进行拦截处理，不需要就交给子View处理。像上面ViewPager的处理方式我们称之为外部拦截法。
2. 内部拦截法：父容器默认不拦截任何事件，如果子元素需要就处理掉，不需要就交给父容器进行处理。这个和android的事件处理机制不一致需要配requestParentDisallowInterceptTouchEvent来进行处理。

题外话：虽然作者把滑动冲突的处理分成了两种方法，但是个人认为这两个方法不应该分开来看。他们的配合使用能够达到比较好的滑动冲突处理效果。requestParentDisallowInterceptTouchEvent（）方法的含义是让父容器不拦截本次事件。细心的同学可能注意到了viewPager在拦截处理滑动事件时就调用了这个方法，让父容器不在拦截这一系列事件。

```
if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                }
```

**所以我个人的结论是：**对于滑动冲突的处理，直接使用符合View的事件分发机制的外部拦截法。辅助内部拦截法实现特殊的需求。想ViewPager中调用这个方法的目的的是在ViewPager一旦进行页面切换它就需要完全处理这次事件，而不是在切换的过程中有可能中断被父容器进行处理。

# 嵌套滑动的实现

关于嵌套滑动的理解：

[https://blog.csdn.net/qq_35561554/article/details/89320881三板斧详解CoordinatorLayo]: 

嵌套滑动&CoordinatorLayout参考过的博客：

https://www.jianshu.com/p/b987fad8fcb4

https://www.jianshu.com/p/f7989a2a3ec2

https://www.jianshu.com/p/7830b05b38bb

https://www.jianshu.com/p/5e6f2ae1d2ec

还有其他一些大佬的文章在写这个篇文章的时候找不到了，十分抱歉。