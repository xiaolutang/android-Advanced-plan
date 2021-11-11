ANR全称 Applicatipon No Response指的是应用程序失去响应，对于ANR的处理很多人的第一反应是通过data/anr/trace.txt下的anr文件配合系统log来进行anr分析。但是有这些消息我们巨额能准确定位anr了吗？

# 什么情况会发生ANR

![](anr发生场景.webp)

需要注意的是Activity的onCreate方法进行耗时操作并不会触发ANR(查看源码并没有找到对应的监测).

# 系统ANR的监控

在 [今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488116&idx=1&sn=fdf80fa52c57a3360ad1999da2a9656b&chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&token=569762407&lang=zh_CN#rd)   中以广播为例介绍了anr的监测过程，我们这里用service来进行举例说明

服务的启动最后会调用ActiveServices#realStartServiceLocked 

```java
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ...
        //发送消息 开机对创建服务进行计时
        bumpServiceExecutingLocked(r, execInFg, "create");
        ...
        //通过ApplicationThread 的代理调用到对应的应用程序 创建服务
        app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
    }
```

ActivityThread#scheduleCreateService()

```java
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            sendMessage(H.CREATE_SERVICE, s);
        }
        
        void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
     
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```

根据[app卡顿系列一 ：Handler同步屏障](https://juejin.cn/post/7028215325627793439) 我们可以知道，此时使用的是同步消息如果当前消息队列存在耗时任务或者大量待处理的消息，有可能导致后续流程不能及时调度。

创建服务的过程最后会由ActivityThread#handleCreateService来进行处理。

```java
private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            //创建Service对象
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //调用Service 生命周期方法 onCreate
            service.onCreate();
            mServices.put(data.token, service);
            try {
                //通知 ActivityManagerService 解除本次anr 检测
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```

ANR的检测与解除检测主要通过bumpServiceExecutingLocked 与serviceDoneExecuting 来完成，实现如下

```java
 private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        ...
        scheduleServiceTimeoutLocked(r.app);
     	...
    }
```

```java
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
```

可以看到服务的anr检测是通过ActivityManagerService的mHandler 发送延时消息来进行处理，如果到了固定时间没有移除消息就会爆ANR。而ActivityManagerService#serviceDoneExecuting 最终会调用ActiveServices#serviceDoneExecutingLocked 来移除对应的message消息。

由此我们可以知道Service的超时逻辑与广播类似：

![](SeriveAnr.webp)

图片来自：[今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488116&idx=1&sn=fdf80fa52c57a3360ad1999da2a9656b&chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&token=569762407&lang=zh_CN#rd)

由此我们可以得出结论：

> 系统服务(AMS，InputService)在将具有超时属性的消息，如创建 Service，Receiver，Input 事件等，通过 Binder 或者其它 IPC 的方式发送到目标进程之后，便启动异步超时监测。而这种性质的监测完全是一种黑盒监测，并不是真的监控发送的消息在真实执行过程中是否超时，也就是说系统不管发送的这个消息有没有被执行，或者真实执行过程耗时有多久，只要在监控超时到来之前，服务端没有接收到通知，那么就判定为超时。

# ANR的捕获

以Service为例，如果没有在指定时间移除ANR监控消息就会触发ActiveServices#serviceTimeout在该方法中如果收集到了ANR原因，会调用AppErrors#appNotResponding

```java
final void appNotResponding(ProcessRecord app, ActivityRecord activity,
            ActivityRecord parent, boolean aboveSystem, final String annotation) {
       //省略代码
       //判断 是不是正在关机  正在处理anr app崩溃 进程被ActivityManagerService杀掉等场景
        synchronized (mService) {
            // PowerManager.reboot() can block for a long time, so ignore ANRs while shutting down.
            if (mService.mShuttingDown) {
                Slog.i(TAG, "During shutdown skipping ANR: " + app + " " + annotation);
                return;
            } else if (app.notResponding) {
                Slog.i(TAG, "Skipping duplicate ANR: " + app + " " + annotation);
                return;
            } else if (app.crashing) {
                Slog.i(TAG, "Crashing app skipping ANR: " + app + " " + annotation);
                return;
            } else if (app.killedByAm) {
                Slog.i(TAG, "App already killed by AM skipping ANR: " + app + " " + annotation);
                return;
            } else if (app.killed) {
                Slog.i(TAG, "Skipping died app ANR: " + app + " " + annotation);
                return;
            }

            // In case we come through here for the same app before completing
            // this one, mark as anring now so we will bail out.
            app.notResponding = true;

            // Log the ANR to the event log.
            EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
                    app.processName, app.info.flags, annotation);

            // Dump thread traces as quickly as we can, starting with "interesting" processes.
            //根据注释内容 尽可能快速的手机anr进行trace信息，把他放在第一个
            firstPids.add(app.pid);

            // Don't dump other PIDs if it's a background ANR
            isSilentANR = !showBackground && !isInterestingForBackgroundTraces(app);
            if (!isSilentANR) {
                int parentPid = app.pid;
                if (parent != null && parent.app != null && parent.app.pid > 0) {
                    parentPid = parent.app.pid;
                }
                if (parentPid != app.pid) firstPids.add(parentPid);

                if (MY_PID != app.pid && MY_PID != parentPid) firstPids.add(MY_PID);

                for (int i = mService.mLruProcesses.size() - 1; i >= 0; i--) {
                    ProcessRecord r = mService.mLruProcesses.get(i);
                    if (r != null && r.thread != null) {
                        int pid = r.pid;
                        if (pid > 0 && pid != app.pid && pid != parentPid && pid != MY_PID) {
                            if (r.persistent) {
                                firstPids.add(pid);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding persistent proc: " + r);
                            } else if (r.treatLikeActivity) {
                                firstPids.add(pid);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding likely IME: " + r);
                            } else {
                                lastPids.put(pid, Boolean.TRUE);
                                if (DEBUG_ANR) Slog.i(TAG, "Adding ANR proc: " + r);
                            }
                        }
                    }
                }
            }
        }

        // Log the ANR to the main log.
    	// 这个就是我们平常在看到的ANR日志信息
        StringBuilder info = new StringBuilder();
        info.setLength(0);
        info.append("ANR in ").append(app.processName);
        if (activity != null && activity.shortComponentName != null) {
            info.append(" (").append(activity.shortComponentName).append(")");
        }
        info.append("\n");
        info.append("PID: ").append(app.pid).append("\n");
        if (annotation != null) {
            info.append("Reason: ").append(annotation).append("\n");
        }
        if (parent != null && parent != activity) {
            info.append("Parent: ").append(parent.shortComponentName).append("\n");
        }

        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

        // don't dump native PIDs for background ANRs unless it is the process of interest
        String[] nativeProcs = null;
        if (isSilentANR) {
            for (int i = 0; i < NATIVE_STACKS_OF_INTEREST.length; i++) {
                if (NATIVE_STACKS_OF_INTEREST[i].equals(app.processName)) {
                    nativeProcs = new String[] { app.processName };
                    break;
                }
            }
        } else {
            nativeProcs = NATIVE_STACKS_OF_INTEREST;
        }

        int[] pids = nativeProcs == null ? null : Process.getPidsForCommands(nativeProcs);
        ArrayList<Integer> nativePids = null;

        if (pids != null) {
            nativePids = new ArrayList<Integer>(pids.length);
            for (int i : pids) {
                nativePids.add(i);
            }
        }

        // For background ANRs, don't pass the ProcessCpuTracker to
        // avoid spending 1/2 second collecting stats to rank lastPids.
    	//收集trace信息
        File tracesFile = ActivityManagerService.dumpStackTraces(
                true, firstPids,
                (isSilentANR) ? null : processCpuTracker,
                (isSilentANR) ? null : lastPids,
                nativePids);

        String cpuInfo = null;
        if (ActivityManagerService.MONITOR_CPU_USAGE) {
            mService.updateCpuStatsNow();
            synchronized (mService.mProcessCpuTracker) {
                cpuInfo = mService.mProcessCpuTracker.printCurrentState(anrTime);
            }
            info.append(processCpuTracker.printCurrentLoad());
            info.append(cpuInfo);
        }

        info.append(processCpuTracker.printCurrentState(anrTime));
		//打印日志
        Slog.e(TAG, info.toString());
        if (tracesFile == null) {
            // There is no trace file, so dump (only) the alleged culprit's threads to the log
            Process.sendSignal(app.pid, Process.SIGNAL_QUIT);
        }

        StatsLog.write(StatsLog.ANR_OCCURRED, app.uid, app.processName,
                activity == null ? "unknown": activity.shortComponentName, annotation,
                (app.info != null) ? (app.info.isInstantApp()
                        ? StatsLog.ANROCCURRED__IS_INSTANT_APP__TRUE
                        : StatsLog.ANROCCURRED__IS_INSTANT_APP__FALSE)
                        : StatsLog.ANROCCURRED__IS_INSTANT_APP__UNAVAILABLE,
                app != null ? (app.isInterestingToUserLocked()
                        ? StatsLog.ANROCCURRED__FOREGROUND_STATE__FOREGROUND
                        : StatsLog.ANROCCURRED__FOREGROUND_STATE__BACKGROUND)
                        : StatsLog.ANROCCURRED__FOREGROUND_STATE__UNKNOWN);
        mService.addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
                cpuInfo, tracesFile, null);

        if (mService.mController != null) {
            try {
                // 0 == show dialog, 1 = keep waiting, -1 = kill process immediately
                int res = mService.mController.appNotResponding(
                        app.processName, app.pid, info.toString());
                if (res != 0) {
                    if (res < 0 && app.pid != MY_PID) {
                        app.kill("anr", true);
                    } else {
                        synchronized (mService) {
                            mService.mServices.scheduleServiceTimeoutLocked(app);
                        }
                    }
                    return;
                }
            } catch (RemoteException e) {
                mService.mController = null;
                Watchdog.getInstance().setActivityController(null);
            }
        }

        synchronized (mService) {
            //省略代码
            // Set the app's notResponding state, and look up the errorReportReceiver
            makeAppNotRespondingLocked(app,
                    activity != null ? activity.shortComponentName : null,
                    annotation != null ? "ANR " + annotation : "ANR",
                    info.toString());

            // Bring up the infamous App Not Responding dialog
            Message msg = Message.obtain();
            msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
            msg.obj = new AppNotRespondingDialog.Data(app, activity, aboveSystem);
			//显示ANR Dialog
            mService.mUiHandler.sendMessage(msg);
        }
    }
```

系统的ANR大致会经历下面的几个流程

1. 判断是不是真的ANR
2. 收集关键进程信息
3. 收集ANR日志
4. dump trace文件
5. 显示ANR Dialog



其中dump trace文件 又可以分为如下流程：

![](ANR_DUMP流程 .webp)

图片来自：[今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488116&idx=1&sn=fdf80fa52c57a3360ad1999da2a9656b&chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&token=569762407&lang=zh_CN#rd)

# ANR原因

经过前面的流程分析，我们知道系统系统监控ANR的原理，总结有下面的原因可能导致解除ANR消息不及时

1. 其它进程/线程抢占CPU 导致主线程不能及时调度任务
2. 在进行调度前有一个或者多个耗时任务
3. 在进行调度的时候主线程正在执行耗时任务
4. 在进行调度的时候主线程有大量待处理的Message