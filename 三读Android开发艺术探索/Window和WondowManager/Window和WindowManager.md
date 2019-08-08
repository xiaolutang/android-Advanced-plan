# Window和WindowManager

今天我们要学习的课程是《Android开发艺术探索》的第8章Window和WindowManager

## 基本概念

Window是Android一个窗口的概念，我们日常开发接触到的机会并不多。但是在某些特殊的情况下我们需要用到它，比如显示一个悬浮按钮。Window是一个抽象类它的具体实现是PhoneWindow。创建一个Window只需要通过WindowManager就可以完成。WindowManager是外界访问Window的入口，Window的具体实现在WindowManagerService中。WindowManager和WindowManagerService交互是一个IPC的过程。Android的所有视图都附着在Window上。

## 通过WindowManager添加Window显示

```kotlin
class WindowAndWindowManagerActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_window_and_window_manager)
        getOverlayPermission()
        addViewToWindow()
    }

    private fun addViewToWindow() {
        val image = ImageView(this)
        image.setImageResource(R.drawable.back)
        image.setBackgroundColor(Color.parseColor("#77d475"))
        val layoutParams = WindowManager.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT,ViewGroup.LayoutParams.WRAP_CONTENT,WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,0,PixelFormat.TRANSPARENT)
        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE or WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
        layoutParams.gravity = Gravity.LEFT or Gravity.TOP
        layoutParams.x = 100
        layoutParams.y = 300
//        val windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        //如何添加到对应的Window中
        windowManager.addView(image,layoutParams)
    }

    //请求悬浮窗权限
    @TargetApi(Build.VERSION_CODES.M)
    private fun getOverlayPermission() {
        //检查是否已经授予权限
        if (Settings.canDrawOverlays(this)) {
            return
        }
        val intent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION)
        intent.data = Uri.parse("package:$packageName")
        startActivityForResult(intent, 0)
    }
}

```

在添加Window时需要注意LayoutParams的几个问题：

1. flag  flag有许多属性书中介绍了几个常见的属性：

   - FLAG_NOT_TOUCHABLE ：这个标记意味着Window不能够获取焦点，也不能接收点击事件。
   - FLAG_NOT_TOUCH_MODAL：这个标记意味着系统会将当期Window区域以外的点击事件传递给底层WIndow,区域内则自己处理。一般而言都需要开启这个标记否则底层的Window无法接收到点击事件。

   系统还有一些其他的Flag可以查阅文档或者在WindowManager的源代码里面进行查看。

2. type type表示Window的类型，Android总体分为3类

   - 应用Window  应用Window对应着Activity
   - 子Window  子Window不能单独存在，它需要附着一个特定的Window,比如Dialog就是一个子Window.
   - 系统Window 系统Window需要声明权限才能创建，比如Toast和系统状态栏就是系统Window。

   系统Window是分层的。应用Window对应（1-99），子Window对应（1000-1999），系统Window对应（1999-2999）。

## Window的内部机制

Window是Android中的抽象概念，每个Window都对应着一个View和ViewRootImpl, Window和View通过ViewRootImpl来建立联系，因此Window并不实际存在，它是以View的形式存在。对Window的访问必须通过WindowManager。

### Window的添加过程

Window的添加需要通过WindowManager的addView来实现，WindowManager是一个真正的接口，它的实现类是WindowManagerImpl,在WindowManagerImpl中Window的三大操作如下

```java
 @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
    
     @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
```

可以发现WindowManagerImpl并没有直接实现Window的三大操作而是将这一切全部交给WindowManagerGlobal来处理。WindowManagerGlobal是全局单例。

WindowManagerGlobal添加View的过程分为如下几部：

1. 检查参数是否合法，如果是子Window还需要调整布局参数

2. 创建ViewRootImpl并将View添加进列表

   WindowManagerGlobal有这样几个重要的列表：

   |             | 作用                         |
   | :---------: | ---------------------------- |
   |   mViews    | 所有Window对应的View         |
   |   mRoots    | 所有Window对应的ViewRootImpl |
   |   mParams   | 所有Window的布局参数         |
   | mDyingViews | 存储正在被删除的对象         |

   代码如下：

   ```java
   root = new ViewRootImpl(view.getContext(), display);
   
               view.setLayoutParams(wparams);
   
               mViews.add(view);
               mRoots.add(root);
               mParams.add(wparams);
   ```

3. 通过ViewRootImpl的setView方法来更新界面并完成Window的添加过程。

4. 在经过一些列的调用后最终由WindowManagerServer来完成Window的添加。

### Window的删除过程

Window的删除过程和添加过程一样都是通WindowManagerGlobal来实现。其实现代码如下：

```java
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```

其逻辑是：先通过findViewLocked找到View的索引，然后调用removeViewLocked来进行删除。

```java
 private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```

在removeViewLocked中会先调用ViewRootImpl的die方法来进行删除。在根据其返回值来决定是否将View放入正在删除的数组。

ViewRootImpl的Die提供同步删除和异步删除，最终他们都会调用到doDie，其中的dispatchDetachedFromWindow是删除View主要实现。dispatchDetachedFromWindow主要实现了如下的功能

1. 垃圾回收相关的工作，比如清除数据和消息、移除回调
2. 通过Seesion的remove方法删除Window。
3. 通知View从Window中移除
4. 移除WindowManagerGlobal维护着的集合中的相关数据。

### Window的更新过程

Window的更新只需要调用WindowManagerGlobal的updateViewLayout方法改变View的LayoutParams即可。

## Window的创建过程



## PopWinodw的本质



