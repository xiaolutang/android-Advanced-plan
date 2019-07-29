[TOC]



# 前言

熟悉android开发的朋友都知道activity可以说是我们日常开发中最常见的一个组件。它承载了与用户的界面交互功能。因此深入了解activity对日常开发会有很大的帮助。今天学习的内容主要来自《Android开发艺术探索》和我自己平时工作中遇到的一些问题详解。

# 原文概述：

基本概念：

1. 典型情况下的生命周期：指在用户参与的情况下，Activity所经过的生命周期的改变。
2. 异常情况下的生命周期：指activity被系统回收或者由于当前设备的Configuration发生改变从而导致Activity被销毁重建。

## 典型情况下的生命周期：

### 生命周期方法详解

参见谷歌官方的一张图片：

![activity_lifecycle](./activity_lifecycle.png)

**onCreate:**表示activity正在被创建，这个是activity的第一个生命周期方法，在这个方法中我们可以做一些初始化工作。如setContentView去加载界面的布局资源，初始化activity所需要的数据。

**onStart:**当onCreate（）退出时，活动进入Started状态，Activity对用户可见。但是还不能进行交互。

**onResume:**Activity进入可以交互的状态，这个时候Activity位于任务栈的顶部，并且能够捕获用户的所有输入，应用程序的大多数核心功能都在onResume中实现。同onStart相比较两者都是出于可见的状态，但是onStart时Activity还在后台，不能交互，onResume时Activity在应用的前台并且可以直接进行交互。

**onPause:**表示activity正在停止，这个时候Activity任然可见，但是不能进行交互。不应该在onPause（）保存应用程序或用户数据，进行网络调用或执行数据库事务。在onPause执行完成后下一个执行onResume还是onStop取决于进入Paused后的状态变化。

**onStop:**当activity不可见的时候会调用这个方法。原因可能是：新的activity开始或activity被销毁。

**onRestart：**当处于Stoped状态的activity重新启动的时候会调用这个方法。这个方法后紧接着会调用onStart 。

### 关于Activity生命周期的几个结论：

1. 针对一个特定的Activity，第一次启动的时候其生命周期方法如下：onCreate -> onStart -> onResume
2. 当用户打开新的activity或回到桌面的时候onPause -> onStop会调用。**特殊情况下（新的activity是透明主题）当前activity不会调用onStop**
3. 当用户再次回到原来的activity(Activity未被销毁的情况下), 回调如下： onRestart -> onStart -> onResume
4. 当用户按返回键回退的时候 ，回调如下：onPause -> onStop -> onDestory
5. activity被系统回收后，重新进入会走新的创建的流程（和1相同）。
6. 对于整个activity的生命周期而言，onCreate和onDestory是配对的，分别标识这创建和销毁，并且只可能有一次调用。onStart和onStop是配对的，onResume和onPause是配对的,他们都有可能多次调用

### 关于典型情况下生命周期的两个问题：

1. onStart、onResume、onPause和onStop本质区别？

   答：onstart和onstop是针对activity是否在前台可见而言来进行回调的。onResume和onPause是针对activity是否在前台可交互而言的，起重点是可交互。

2. 当前的activity A 启动Activity B他们整体的生命周期方法是如何回调的？

   答：A:onPause -> B:onCreate -> onStart ->  B:onResume -> A:onStop

## 异常情况下的生命周期：

### 资源相关的系统配置发生改变导致Activity被杀死并重建

默认情况下系统配置发生改变，activity会被销毁并重新创建建。其onPause,onStop和onDestory都会被调用。同时由于系统是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前的数据。同时在重新创建时系统会把onSaveInstanceState方法所保存的数据传递给onCreate和onRestoreInstanceState我们可以在这个时候恢复数据。

### 资源内存不足导致优先级低的activity被杀死

activity优先级：

1. 前台activity-正在和用户交互，优先级最高
2. 可见但非前台activity,比如activity中弹出了一个对话框，导致activity可见但是位于后台无法和activity交互
3. 后台activity:已经执行了onStop的activity

### 设置在系统某些资源发生改变的时候不重新创建activity

```xml
android:configChanges="orientation"
```

上面配置了屏幕旋转的时候activity不重建。如果有多个配置使用 | 连接，同时在对应的配置属性改变时会调用onConfigutationChanged。

## Android资源配置改变对于activti而言发生了什么？

在ActivityRecord中有这样一个方法：从代码的注释可以看出这里决定了activity是销毁重建还是保存

```java
/**
     * Make sure the given activity matches the current configuration. Returns false if the activity
     * had to be destroyed.  Returns true if the configuration is the same, or the activity will
     * remain running as-is for whatever reason. Ensures the HistoryRecord is updated with the
     * correct configuration and all other bookkeeping is handled.
     */
boolean ensureActivityConfigurationLocked(int globalChanges, boolean preserveWindow) {
    //省略部分代码
    if (shouldRelaunchLocked(changes, mTmpConfig) || forceNewConfig) {
        //销毁activity
    }
    //调用activity的配置改变
    // Default case: the activity can handle this new configuration, so hand it over.
        // NOTE: We only forward the override configuration as the system level configuration
        // changes is always sent to all processes when they happen so it can just use whatever
        // system level configuration it last got.
        if (displayChanged) {
            scheduleActivityMovedToDisplay(newDisplayId, newMergedOverrideConfig);
        } else {
            scheduleConfigurationChanged(newMergedOverrideConfig);
        }
        stopFreezingScreenLocked(false);
}
```

如果最后不销毁activity调用配置更改的方法最后会调用到ActivityThread的相关配置来进行处理。在API 27 的ActivityThread 中有一个方法 performActivityConfigurationChanged这个方法决定了是重建activity还是调用activity。接下来我们看看关键源码：

   

```java
 /**
     * Decides whether to update an Activity's configuration and whether to inform it.
     * @param activity The activity to notify of configuration change.
     * @param newConfig The new configuration.
     * @param amOverrideConfig The override config that differentiates the Activity's configuration
     *                         from the base global configuration. This is supplied by
     *                         ActivityManager.
     * @param displayId Id of the display where activity currently resides.
     * @param movedToDifferentDisplay Indicates if the activity was moved to different display.
     * @return Configuration sent to client, null if no changes and not moved to different display.
     */
    private Configuration performActivityConfigurationChanged(Activity activity,
            Configuration newConfig, Configuration amOverrideConfig, int displayId,
            boolean movedToDifferentDisplay) {
        //标记位，是否更新改变
        boolean shouldChangeConfig = false;
        if (activity.mCurrentConfig == null) {
            shouldChangeConfig = true;
        } else {
            // If the new config is the same as the config this Activity is already running with and
            // the override config also didn't change, then don't bother calling
            // onConfigurationChanged.
            //下面这句代码的意思是，对比当前在activity运行的配置和更新的配置，从上面的英文注释也可以明白，如果配相同就没必要调用onConfigrationChanged
            final int diff = activity.mCurrentConfig.diffPublicOnly(newConfig);

            if (diff != 0 || !mResourcesManager.isSameResourcesOverrideConfig(activityToken,
                    amOverrideConfig)) {
                // Always send the task-level config changes. For system-level configuration, if
                // this activity doesn't handle any of the config changes, then don't bother
                // calling onConfigurationChanged as we're going to destroy it.
                //下面的条件判断关注： (~activity.mActivityInfo.getRealConfigChanged() & diff) 忽略其余的两个条件 其本质就是对比当前的改变项是否全部在manifest的configChanges中进行了属性声明，如果改变项全部都在其中进行了声明那么就会调用onConfigrationed
                if (!mUpdatingSystemConfig
                        || (~activity.mActivityInfo.getRealConfigChanged() & diff) == 0
                        || !REPORT_TO_ACTIVITY) {
                    shouldChangeConfig = true;
                }
            }
        }
        //注意着里两个参数  shouldChangeConfig 是标志是说明是否在manifest中进行了属性声明的，movedToDifferentDisplay 参考参数的注释简单来就理解就是是不是更换了显示屏幕。在我们手机上的这个值一直是false, 这个时如果没有在manifest进行属性说明的话，这个方法到这里就返回null  根据函数说明大胆猜测返回null系统会销毁当前activity
        if (!shouldChangeConfig && !movedToDifferentDisplay) {
            // Nothing significant, don't proceed with updating and reporting.
            return null;
        }

        // Propagate the configuration change to ResourcesManager and Activity.

        // ContextThemeWrappers may override the configuration for that context. We must check and
        // apply any overrides defined.
        Configuration contextThemeWrapperOverrideConfig = activity.getOverrideConfiguration();

        // We only update an Activity's configuration if this is not a global configuration change.
        // This must also be done before the callback, or else we violate the contract that the new
        // resources are available in ComponentCallbacks2#onConfigurationChanged(Configuration).
        // Also apply the ContextThemeWrapper override if necessary.
        // NOTE: Make sure the configurations are not modified, as they are treated as immutable in
        // many places.
        final Configuration finalOverrideConfig = createNewConfigAndUpdateIfNotNull(
                amOverrideConfig, contextThemeWrapperOverrideConfig);
        mResourcesManager.updateResourcesForActivity(activityToken, finalOverrideConfig,
                displayId, movedToDifferentDisplay);

        activity.mConfigChangeFlags = 0;
        activity.mCurrentConfig = new Configuration(newConfig);

        // Apply the ContextThemeWrapper override if necessary.
        // NOTE: Make sure the configurations are not modified, as they are treated as immutable
        // in many places.
        final Configuration configToReport = createNewConfigAndUpdateIfNotNull(newConfig,
                contextThemeWrapperOverrideConfig);

        if (!REPORT_TO_ACTIVITY) {
            // Not configured to report to activity.
            return configToReport;
        }

        if (movedToDifferentDisplay) {
            activity.dispatchMovedToDisplay(displayId, configToReport);
        }

        if (shouldChangeConfig) {
            activity.mCalled = false;
            activity.onConfigurationChanged(configToReport);
            if (!activity.mCalled) {
                throw new SuperNotCalledException("Activity " + activity.getLocalClassName() +
                                " did not call through to super.onConfigurationChanged()");
            }
        }

        return configToReport;
    }

    public final void applyConfigurationToResources(Configuration config) {
        synchronized (mResourcesManager) {

```

通过阅读源码有如下的几个小结论（API27）：

1. 在manifest中如果不对相关的改变进行声明。系统对不同状态的activity有不同的处理，对于Resume的activity会销毁重建（和书中说的有些不同书中没有针对不同状态进行说明，我这里也感觉对这个详细说明意义不大）。

2. 自从Android 3.2（API 13），在设置Activity的android:configChanges="orientation|keyboardHidden"后，还是一样会重新调用各个生命周期的。因为screen size也开始跟着设备的横竖切换而改变。所以，在AndroidManifest.xml里设置的MiniSdkVersion和 TargetSdkVersion属性大于等于13的情况下，如果你想阻止程序在运行时重新加载Activity，除了设置"orientation"，你还必须设置"ScreenSize"

   ```java
   /**
        * @hide
        * Unfortunately some developers (OpenFeint I am looking at you) have
        * compared the configChanges bit field against absolute values, so if we
        * introduce a new bit they break.  To deal with that, we will make sure
        * the public field will not have a value that breaks them, and let the
        * framework call here to get the real value.
        */
       public int getRealConfigChanged() {
           return applicationInfo.targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB_MR2
                   ? (configChanges | ActivityInfo.CONFIG_SCREEN_SIZE
                           | ActivityInfo.CONFIG_SMALLEST_SCREEN_SIZE)
                   : configChanges;
       }
      
   ```

3. 在资源配置发生改变的时候是先判断是不是需要销毁，在判断是否调用onConfigrationChanged。

# 关于资源配置改变的应用（猜想）

通过资源配置实现app换肤，改变字体的功能？

