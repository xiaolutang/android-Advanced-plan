说到自定义View,可能大多数人都知道自定义View有三大流程。测量，布局，绘制。但是对于详细的过程却不是特别清楚。比如：

1. 为什么直接继承View为什么wrap_content会失效？
2. 父View是如何对子View进行测量的？
3. 子View的MeasureSpec是如何决定的？
4. 为什么在onMeasure中不调用setMeasuredDimension会发生异常。

本文将致力于两个位置。

1. 关键方法的调用流程
2. 对于重要的关键细节进行详细说明



# 

# 认识MeasureSpec

MeasureSpec 是View测量的辅助类，View的MeasureSpec由自身的宽高和父容器共同决定。系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec。

# 测量流程

View的测量、布局、绘制都发起于ViewRootImpl。我们这里不关心这一个其实流程直接进入与我们密切相关View&ViewGroup的测量流程。View的测量通过两个方法来完成，measure和onMeasure。measure被设置成final，意味着子View不能继承它，自身的独特测量逻辑需要通过重写onMeasure来实现。

View#measure过程

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        //省略 代码
        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

可以看到View自身的测量逻辑相对还是比较简单，调用onMeasure测量自生，然后检查PFLAG_MEASURED_DIMENSION_SET标记是否有设置，如果没有设置抛出异常，这就是为什么必须调用setMeasuredDimension。

View的onMeasure实现如下：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

可以看到如果直接继承View,使用wrap_content与match_parent是相同的效果。



我们知道Activity的根布局是DecorView，View的测量发起于measure方法

