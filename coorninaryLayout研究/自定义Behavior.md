自定义behavior主要有两种用途：

1. 实现嵌套滑动
2. 依赖于某个view进行对应的变化

当然behavior肯定不止这两个功能

简单的实现天猫顶部title的效果图如下：![](C:\Users\txl\Desktop\天猫效果.png)

先来个效果图进行对比：功能效果比较简陋主要是为了学习自己定义behavior。

![tian_mao](.\tian_mao.gif)

![custom_tian_mao](.\custom_tian_mao.gif)

具体的实现思路是：

- 首先我们事先嵌套滑动的HeaderBehavior:

  实现HeaderBehavior: 主要功能是在嵌套滑动的时候处理滑动事件，两个逻辑：

  1. 在向上滑动的时候优先滑动header让它进上滑变化
  2. 在向下滑动时滞后滑动header下拉变化

  明白了这个对应的实现代码如下：

  ```
  /**
   * 当发起嵌套滑动的时候处理
   * */
  class HeaderBehavior : CoordinatorLayout.Behavior<View> {
  
      val TAG = HeaderBehavior::class.java.simpleName
  
      var mMaxTranslationY = 0
      var mLeftMargin = 0
      var mRightMargin = 0
  
      /**
       * y方向上的平移
       * */
      var mTranslationY = 0f
  
      private var isInit = false
  
  
      /**
       * y方向上滑动距离的变化距离比例，
       * */
      private val dyTransformationSpeed = 2
  
      internal fun init(context: Context) {
          mMaxTranslationY = DisplayUtil.dip2px(context, 55f)
          mLeftMargin = DisplayUtil.dip2px(context, 45f)
          mRightMargin = DisplayUtil.dip2px(context, 45f)
      }
  
      constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
  
      constructor() : super()
  
      override fun onStartNestedScroll(coordinatorLayout: CoordinatorLayout, child: View, directTargetChild: View, target: View, axes: Int, type: Int): Boolean {
          if (!isInit) {
              init(child.context)
              isInit = true
          }
          if (axes and ViewCompat.SCROLL_AXIS_VERTICAL != 0) {
              return true
          }
          return false
      }
  
  
      override fun onNestedPreScroll(coordinatorLayout: CoordinatorLayout, child: View, target: View, dx: Int, dy: Int, consumed: IntArray, type: Int) {
          mTranslationY = child.translationY
          if (dy > 0) {//向上
              //向上滑动优先滑动顶部的header
              if (Math.abs(mTranslationY) + dy < mMaxTranslationY) {
                  consumed[1] = dy
              } else if (Math.abs(mTranslationY) < mMaxTranslationY && Math.abs(mTranslationY) + dy > mMaxTranslationY) {
                  consumed[1] = (mMaxTranslationY - Math.abs(mTranslationY)).toInt()
              }
              child.translationY = mTranslationY - consumed[1] / dyTransformationSpeed
              transformationView(child)
          }
      }
  
      override fun onNestedScroll(coordinatorLayout: CoordinatorLayout, child: View, target: View, dxConsumed: Int, dyConsumed: Int, dxUnconsumed: Int, dyUnconsumed: Int, type: Int) {
          mTranslationY = child.translationY
          if (dyUnconsumed < 0) {//向下
              //在向下滑动时，当需要滑动的view不在消耗这个滑动的距离将header下拉
              if (Math.abs(mTranslationY) > 0 && mTranslationY - dyUnconsumed <= 0) {
                  child.translationY = mTranslationY - dyUnconsumed / dyTransformationSpeed
              } else if (Math.abs(mTranslationY) > 0 && mTranslationY - dyUnconsumed > 0) {
                  child.translationY = 0f
              }
              transformationView(child)
          }
      }
  
      override fun onNestedPreFling(coordinatorLayout: CoordinatorLayout, child: View, target: View, velocityX: Float, velocityY: Float): Boolean {
          var handle = false
          if (velocityY > 0) {//向下
  
          } else {//向上
  
          }
          return handle
      }
  
      override fun onNestedFling(coordinatorLayout: CoordinatorLayout, child: View, target: View, velocityX: Float, velocityY: Float, consumed: Boolean): Boolean {
          return super.onNestedFling(coordinatorLayout, child, target, velocityX, velocityY, consumed)
      }
  
      /**
       * 对当前使用这个view的behavior进行变化处理
       * */
      private fun transformationView(child: View, directUp:Boolean = false) {
          val tvSearch = child.findViewById<TextView>(R.id.tv_transformation)
          val translationY = Math.abs(child.translationY)
          val ratio = translationY / mMaxTranslationY
          //我的布局是使用的FrameLayout
          val layoutParams = tvSearch.layoutParams as FrameLayout.LayoutParams
          layoutParams.marginStart = (mLeftMargin * ratio).toInt()
          layoutParams.marginEnd = (mRightMargin * ratio).toInt()
          tvSearch.layoutParams = layoutParams
  
          child.background.mutate().alpha = (255 * ratio).toInt()
      }
  }
  ```

- 其次实现HeaderDependsOnBehavior，依赖于header View实现对header变化的监听

```
**
 * 依赖于header变化独自生进行处理
 * */
class HeaderDependsOnBehavior : CoordinatorLayout.Behavior<View> {
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)

    constructor() : super()

    override fun layoutDependsOn(parent: CoordinatorLayout?, child: View?, dependency: View?): Boolean {
        return isDependsOn(dependency)
    }

    private fun isDependsOn(dependency: View?): Boolean {
        return dependency?.id == R.id.frame_header
    }

    override fun onDependentViewChanged(parent: CoordinatorLayout?, child: View?, dependency: View?): Boolean {
        child?.translationY = dependency!!.translationY + dependency.height
        return super.onDependentViewChanged(parent, child, dependency)
    }

    override fun onDependentViewRemoved(parent: CoordinatorLayout?, child: View?, dependency: View?) {
        child?.translationY = 0f
    }
}
```

这样我们完成了对behavior的实现。

那么我们该如何引用我们自己的behavior呢？

通过源码我们知道实现设置behavior有四种方式：

1. 通过CoordinatorLayout.LayoutParams 的setBehavior进行设置

2. 通过xml布局的layout_behavior进行设置

   这个可以使类的全路径。也可以是相对路径

   ![1556602668639](C:\Users\txl\AppData\Roaming\Typora\typora-user-images\1556602668639.png)

   ![1556602712225](C:\Users\txl\AppData\Roaming\Typora\typora-user-images\1556602712225.png)

3. 对特定的VIew实现AttachedBehavior接口指定behavior

   

4. 通过注解的方式在对应的view上指定：可以参考

   ![1556601824583](C:\Users\txl\AppData\Roaming\Typora\typora-user-images\1556601824583.png)

他们的优先级分别是 1>2>3>4 即通过setBehavior的方式设置behavior的优先级最高，通过注解的方式优先级最低。

完成的demo代码：

activity:

```
class TianMaoBehaviorDemoActivity : AppCompatActivity() {
    var recyclerView:RecyclerView? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_tian_mao_behavior_demo)
        initView()
    }

    private fun initView() {
        recyclerView = findViewById(R.id.recycler_view)
        recyclerView?.layoutManager = LinearLayoutManager(this,LinearLayoutManager.VERTICAL,false)
        var adapter = MyAdapter(this)
        val list = ArrayList<String>()
        for (i in 0..20){
            list.add("我是第 $i 个元素")
        }
        adapter.addListData(list)
        recyclerView?.adapter = adapter
    }

    class MyAdapter(context: Context) : BaseAdapter<String>(context){
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BaseViewHolder<String> {
            return MyViewHolder(layoutInflater.inflate(R.layout.textview_item_height_60dp,parent,false))
        }
    }

    class MyViewHolder(itemView: View) : BaseAdapter.BaseViewHolder<String>(itemView){
        override fun onBindViewHolder(position: Int, data: String, adapter: BaseAdapter<String>) {
            super.onBindViewHolder(position, data, adapter)
            itemView.setBackgroundColor(ColorUtils.randomColor())
            itemView as TextView
            itemView.text = data
        }
    }
}

```

activity布局：activity_tian_mao_behavior_demo.xml

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".coordinatorlayout.TianMaoBehaviorDemoActivity">
    <android.support.v7.widget.RecyclerView
        android:id="@+id/recycler_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior=".coordinatorlayout.behavior.HeaderDependsOnBehavior"/>
    <FrameLayout
        android:id="@+id/frame_header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/colorPrimary"
        app:layout_behavior=".coordinatorlayout.behavior.HeaderBehavior">
        <TextView
            android:id="@+id/tv_tm_title"
            android:layout_width="match_parent"
            android:layout_height="45dp"
            android:text="天猫 TMALL"
            android:gravity="center" />
        <TextView
            android:id="@+id/tv_transformation"
            android:layout_width="match_parent"
            android:layout_height="45dp"
            android:text="我是搜索"
            android:gravity="center"
            android:layout_marginTop="55dp"
            android:background="@android:color/white"/>
    </FrameLayout>
    <ImageView
        android:layout_width="45dp"
        android:layout_height="45dp"
        android:layout_gravity="start"
        android:src="@android:drawable/ic_menu_camera"
        android:scaleType="centerCrop"/>
    <ImageView
        android:layout_width="45dp"
        android:layout_height="45dp"
        android:layout_gravity="end"
        android:src="@android:drawable/ic_menu_search"
        android:scaleType="centerCrop"/>
</android.support.design.widget.CoordinatorLayout>
```

这样一个简陋的类似天猫顶部效果的behavior就完成了。这里主要是为了理解自定义behavior的原理就没有完全实现天猫的效果。