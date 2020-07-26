# 概述

LiveData是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。

如果观察者（由 Observer 类表示）的生命周期处于 STARTED 或 RESUMED 状态，则 LiveData 会认为该观察者处于活跃状态。LiveData 只会将更新通知给活跃的观察者。为观察 LiveData 对象而注册的非活跃观察者不会收到更改通知。

您可以注册与实现 LifecycleOwner 接口的对象配对的观察者。有了这种关系，当相应的 Lifecycle 对象的状态变为 DESTROYED 时，便可移除此观察者。 这对于 Activity 和 Fragment 特别有用，因为它们可以放心地观察 LiveData 对象而不必担心泄露（当 Activity 和 Fragment 的生命周期被销毁时，系统会立即退订它们）。

# 使用LiveData的优势

- 确保界面符合数据状态
- 不会发生内存泄漏
- 不会因为activity停止而崩溃
- 不在需要手动处理生命周期
- 数据始终保持最新状态
- 共享资源

# 使用LiveData

使用LiveDate非常简单，大致有下面3个步骤

1. 创建LiveData例以存储某种类型的数据。这通常在ViewModel中完成。
2. 创建Observer对象，该方法可以控制当 `LiveData` 对象存储的数据更改时会发生什么。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中创建 `Observer` 对象
3. 将Observer对象与LiveData关联。
4. 更新LiveData中的

简单示例：使用LiveData来存储一个数字，在界线上显示这个数字

```kotlin
class NumViewModel:ViewModel() {
    val numData  = MutableLiveData<Int>()
}
```

```kotlin
class LiveDataActivity : AppCompatActivity() {
    private lateinit var numViewModel:NumViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
        numViewModel = ViewModelProviders.of(this).get(NumViewModel::class.java)
        val obs = object: Observer<Int>{
            override fun onChanged(t: Int?) {
                tv_text.text = "$t"
            }
        }
        numViewModel.numData.observe(this,obs)
        button_sub.setOnClickListener {
            val num = numViewModel.numData.value
            if(num == null){
                numViewModel.numData.value = 0
            }else{

                numViewModel.numData.value = num - 1
            }
        }
        button_add.setOnClickListener {
            val num = numViewModel.numData.value
            if(num == null){
                numViewModel.numData.value = 0
            }else{

                numViewModel.numData.value = num + 1
            }
        }
    }
}
```

LiveData的更新数据有两种当时 setValue和postValue。在主线程使用setValue在子线程使用postValue。

# 源码阅读

我们发现LiveData的使用非常简单，作为一个有追求的程序员，我们不能仅仅停留在使用阶段。接下来我们将通过源码阅读来看看liveData的实现原理。

LiveData的源码整体上并不复杂，整个LiveData类加上注释也不到500行代码。我们先来看看添加监听的代码：

```java
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
    //如果当前的LifecycleOwner处于destory就直接不添加了
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    //检测同一个监听者是不是从不同的LifecycleOwner添加进来
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
    //本质上是通过感知Lifecycle的生命周期变化，来决定如何处理
        owner.getLifecycle().addObserver(wrapper);
    }
```

从添加监听的逻辑我们可以看到，为LiveData添加监听，最后是通过给Lifecycle添加监听来完成。并且LiveData通过LifecycleBoundObserver类来对我们的监听对象进行装饰，从而加强监听者的功能。

```java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
            //当LifecycleOwner处于destory的时候直接移除监听，从而避免了内存泄漏和不必要的回调
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }
```

根据前面我们阅读Lifecycle我们知道onStateChanged方法会多次调用，但是LiveData在一次数据变化期间只会给到一次回调。我们 来看看它具体是怎么实现的。可以明显看到当生命周期处于destory的时候会直接将监听移除。在activeStateChanged方法中传递的参数是由

shouldBeActive返回的，这个方法可以简单的理解为 当前处于onResume或onStart状态的时候返回true,否则返回false。

```java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {//新传入的状态与上一次的状态相同，不进行处理
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        onActive();
    }
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
    }
    if (mActive) {//分发数据改变
        dispatchingValue(this);
    }
}
```

dispatchingValue的逻辑非常简单根据传入对象是否为空来判断是当前对象的接收数据改变还是所有的监听者都接收数据改变。

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                //真正的分发处理
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
```

我们再来看看数据改变分发的具体实现：

```java
private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {//检查当前监听者维护的状态
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {//检查当前生命周期组件的状态是否可用
            observer.activeStateChanged(false);
            return;
        }
    //更新版本值判断，是否设置了新的数据需要监听变化
        if (observer.mLastVersion >= mVersion) {
            return;
        }
    	//更新版本值，通知数据改变
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged((T) mData);
    }
```

# 小结：

- LiveData底层通过Lifecycle来监听activity或fragment的生命周期变化，在destory的时候自动移除监听，减少了内存泄漏的 发生
- 通过持有LifecycleOwner，在进行数据分发的时候获取当前的生命周期状态来决定是否分发数据改变。
- 每一次数据改变都会增加mVersion的值，防止因为生命周期的变化而不断调用同一个监听者的onDataChange方法。
- 从设计模式上来看的话livedata用到了观察者模式和装饰模式。