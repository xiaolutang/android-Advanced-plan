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

   系统Window是分层的。应用Window对应（1-99），子Window对应（1000-1999），系统Window对应（1999-2999）