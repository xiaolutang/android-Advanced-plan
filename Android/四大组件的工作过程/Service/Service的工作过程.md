网上其实有很多关于Service的博客，但是为什么要写Service?

原因有下面几点：

1. 通过这次学习输出让自己加深对Service的了解
2. 本篇文章希望通过源码的分析，来解释Service的生命周期现象
3. 对于服务工作实现的细节上，有很多自己不明白的地方。但是网上也并没有一个详细的解答

# 问题

1. startService时为什么onCreate只调用一次，而onStartCommand却能够调用多次？
2. bindService的过程中为什么onBind只会在第一次绑定的过程中调用？
3. 为什么startService加bindService开启的服务需要同时调用stop和unbind来暂停？
4. 服务是如何检测anr的？
5. 绑定服务之后，服务端是如何调用到客户端的？
6. ServiceConnection#onServiceConnected在哪个线程执行？

如果你对上面的问题了然于心，那么这篇文章对你的作用可能不大。如果你对此还有疑惑，希望这篇文章能够给到一些小的帮助。



文章出下面的几个方向出发，进行源码探究，以解释上面提出的问题

- 服务的生命周期
- 服务的工作过程
- 服务的anr检测

# 服务的生命周期

服务主要有下面的几个方法

![image-20210707102033288](Service生命周期方法.png)

即

onCreate

onStartCommand:

onBind

onRebind

onUnbind

onDestroy

关于每个生命周期方法的作用，这里不详细说明，可以参考网上的博客或者直接看官方文档。

https://developer.android.google.cn/guide/components/services

服务可以通知startService或者bindService启动它们的生命周期如下：

![](service生命周期流程.png)



- 使用startService（）开启服务时，onCreate（）只会调用一次，onStartCommand（）的调用次数与startService()的调用次数相同。startService开启的服务需要通过 stopService()或者Service自己调用stopself()来停止服务。服务停止后会调用onDestory()
- 使用bindService()来开启的服务 onCreate 和 onBind()都只会调用一次（不论bindService调用多少次），bindService开启的服务需要通过 onBind来解绑暂停，解绑后会依次调用onUnbind()和unDestory()。需要注意的是：需要服务没有和任何一个客户端绑定之后才会调用onUnbind和onDestory暂停服务。
- 同时使用startService（）和 bindService（）会先调用 onCreate() 、onStartCommand() /onBind()  两者的调用关系取决于先调用的是startService还是bindService。暂停服务，需要调用stopService和unBindService同时调用。
- 关于onRebind  onRebind调用有几个前提条件。
  - 同时使用startService和bindService
  - 服务必须先绑定，然后绑定的客户端减少到0
  - onUnbind 返回true。

关于生命周期实验demo地址：

https://github.com/xiaolutang/androidTool/blob/master/app/src/main/java/com/example/txl/tool/service/ServiceDemoActivity.java



# 服务的工作过程

看个图片舒缓一下，接下来进入枯燥的源码阅读

![](休息图片.jpg)

服务的工作过程主要是应用程序，与ActivityManagerService的交互过程。代码分析基于api 30

## startService的过程

我们知道四大组件的很多工作都最终执行的位置都是在ContextImpl来万层，启动服务也不例外。Activity#startService()最终会调用ContextImpl#startService()

```java
 @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            //检测启动目标的有效性，主要是目标包名和启动类不能同时为空。
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            //通过binder跨进程调用到ActivityManagerService
            ComponentName cn = ActivityManager.getService().startService(
                    mMainThread.getApplicationThread(), service,
                    service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                    getOpPackageName(), getAttributionTag(), user.getIdentifier());
            ...
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```

ActivityManagerService最终会调用到ActiveServices#bringUpServiceLocked

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
    //根据 ServiceRecord#app判断当前服务是不是在运行，如果service在运行就调用sendServiceArgsLocked传递参数
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }
		...
       //真正的启动服务的位置
       if (app != null && app.thread != null) {
            realStartServiceLocked(r, app, execInFg);
       }
      ...
          if (app == null && !permissionsReviewRequired) {
            // TODO (chriswailes): Change the Zygote policy flags based on if the launch-for-service
            //  was initiated from a notification tap or not.
              //应用程序不存在，先创建应用程序
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
		...
        return null;
    }
```

ActiveServices#realStartServiceLocked

```java
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
    //这里给ServiceRecord#app赋值，表示当前服务已经启动
        r.setProcess(app);
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        //注意这个东西，这个是为了监测anr而发送一个延时消息
        bumpServiceExecutingLocked(r, execInFg, "create");
        ...
            //回调服务所在应用程序，创建服务
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
    	...
            //回调服务所在应用程序传递参数
             sendServiceArgsLocked(r, execInFg, true);
    	...
    }
```

### onCreate()调用

ApplicationThread#scheduleCreateService方法通过handler H 进行消息转发，最终会调用到ActivityThread#handleCreateService

ActivityThread#handleCreateService

```java
 private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
			//创建context
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            //获取application
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            //创建service
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            // Service resources must be initialized with the same loaders as the application
            // context.
            context.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            context.setOuterContext(service);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //回调service onCreate
            service.onCreate();
            mServices.put(data.token, service);
            try {
                //发送消息，结束anr检测
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

### onStartCommand（）的调用

参数信息的发送是通过ActiveServices#sendServiceArgsLocked（）来进行的。

```java
 private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();//在star
        if (N == 0) {
            return;
        }
        ...
        bumpServiceExecutingLocked(r, execInFg, "start");
        ...
         r.app.thread.scheduleServiceArgs(r, slice);
        ...
    }
```

与onCreate的调用过程类似，scheduleServiceArgs 也会经过H转发 最终会调用ActivityThread#handleServiceArgs

```java
private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {
                //关键调用 onStartCommand
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }

                QueuedWork.waitToFinish();

                try {
                    //结束anr监测
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to start service " + s
                            + " with " + data.args + ": " + e.toString(), e);
                }
            }
        }
    }
```



启动服务会经历如下过程

1. 判断服务所在进程是不是存在，不存在则先启动对应的进程
2. 判断服务是不是已经在运行，运行则调用ActiveServices#sendServiceArgsLocked传递参数，否则调用ActiveServices#realStartServiceLocked 先创建服务，在传递参数。
3. 服务不论是在创建还是传递参数的过程中都会调用ActiveServices#bumpServiceExecutingLocked 来发送消息监测anr。（所有生命周期方法都有这个特性呢？）

## stopService的过程

暂定服务有两种方式，调用stopService(),服务自己调用stopself()。我们这里分析stopService的过程。

stopService的最终会调用ActiveServices#stopServiceLocked（）

```java
private void stopServiceLocked(ServiceRecord service) {
       ...
        service.startRequested = false;
        ...
        service.callStart = false;
		//判断是否需要暂停服务
        bringDownServiceIfNeededLocked(service, false, false);
    }

private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
            boolean hasConn) {
        //Slog.i(TAG, "Bring down service:");
        //r.dump("  ");
		//判断服务是否需要保存，判断依据是当前是否有绑定的对象 以及service.startRequested 是否为true
        if (isServiceNeededLocked(r, knowConn, hasConn)) {
            return;
        }

        // Are we in the process of launching?
        if (mPendingServices.contains(r)) {
            return;
        }

        bringDownServiceLocked(r);
    }


```

bringDownServiceLocked的实现过程

```java
 private final void bringDownServiceLocked(ServiceRecord r) {
        //Slog.i(TAG, "Bring down service:");
        //r.dump("  ");

        // Report to all of the connections that the service is no longer
        // available.
     	//如果此时还有连接的服务，通知这些连接不可用
        ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
        for (int conni = connections.size() - 1; conni >= 0; conni--) {
            ArrayList<ConnectionRecord> c = connections.valueAt(conni);
            for (int i=0; i<c.size(); i++) {
                ConnectionRecord cr = c.get(i);
                // There is still a connection to the service that is
                // being brought down.  Mark it as dead.
                cr.serviceDead = true;
                cr.stopAssociation();
                try {
                    cr.conn.connected(r.name, null, true);
                } catch (Exception e) {
                    Slog.w(TAG, "Failure disconnecting service " + r.shortInstanceName
                          + " to connection " + c.get(i).conn.asBinder()
                          + " (in " + c.get(i).binding.client.processName + ")", e);
                }
            }
        }

        // Tell the service that it has been unbound.
     //如果此时还有连接的服务，解绑这些连接
        if (r.app != null && r.app.thread != null) {
            boolean needOomAdj = false;
            for (int i = r.bindings.size() - 1; i >= 0; i--) {
                IntentBindRecord ibr = r.bindings.valueAt(i);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                        + ": hasBound=" + ibr.hasBound);
                if (ibr.hasBound) {
                    try {
                        bumpServiceExecutingLocked(r, false, "bring down unbind");
                        needOomAdj = true;
                        ibr.hasBound = false;
                        ibr.requested = false;
                        r.app.thread.scheduleUnbindService(r,
                                ibr.intent.getIntent());
                    } catch (Exception e) {
                        Slog.w(TAG, "Exception when unbinding service "
                                + r.shortInstanceName, e);
                        needOomAdj = false;
                        serviceProcessGoneLocked(r);
                        break;
                    }
                }
            }
            if (needOomAdj) {
                mAm.updateOomAdjLocked(r.app, true,
                        OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
            }
        }
		...
        // 停止服务
         r.app.thread.scheduleStopService(r);
		...
     //清除绑定信息
        if (r.bindings.size() > 0) {
            r.bindings.clear();
        }
    }
```



## bindService的过程

我们知道绑定服务时传递的ServiceConnection对象并不具备跨进程传输的能力，那么绑定服务的过程中他是如何工作的呢？

在ContentxImpl中bindService最终会调用到bindServiceCommon 它的实现如下：

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            String instanceName, Handler handler, Executor executor, UserHandle user) {
        // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
    //该对象是进行跨进程传输的队形，当绑定成功过后通过它进行回调
        IServiceConnection sd;
        ...
        if (mPackageInfo != null) {
            if (executor != null) {
                //getOuterContext() 意味着获取的是activiy、service、或者application  特别的当activity活这个service结束时，可以对绑定的服务进行清理，从系统级别防止内存泄漏
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
            } else {
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess(this);
            int res = ActivityManager.getService().bindIsolatedService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

在绑定服务的过程中会先调用LoadedApk#getServiceDispatcher()获取IServiceConnection对象，然后在通过ActivityManagerService进行绑定

### IServiceConnection的创建过程

getServiceDispatcher最终会调用到LoadedApk#getServiceDispatcherCommon（）

```java
private IServiceConnection getServiceDispatcherCommon(ServiceConnection c,
            Context context, Handler handler, Executor executor, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
                sd = map.get(c);
            }
            if (sd == null) {
                if (executor != null) {
                    sd = new ServiceDispatcher(c, context, executor, flags);
                } else {
                    sd = new ServiceDispatcher(c, context, handler, flags);
                }
                if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler, executor);
            }
            return sd.getIServiceConnection();
        }
    }
```

代码的逻辑很简单，以context作为key存储了一个类型为 ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>  的map,然后依据 ServiceConnection 返回LoadedApk.ServiceDispatcher#getIServiceConnection()对应的东西。

LoadedApk#ServiceDispatcher

```java
 static final class ServiceDispatcher {
    private final ServiceDispatcher.InnerConnection mIServiceConnection;
     IServiceConnection getIServiceConnection() {
            return mIServiceConnection;
        }
 }
```

可以看到其实返回的是内部类ServiceDispatcher.InnerConnection。



### ActivityManagerService绑定的过程

绑定服务会调用ActivityManagerService#bindIsolatedService方法在里面又会调用ActiveServices#bindServiceLocked方法

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String instanceName, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        ...

            //以进程为单位存储 绑定服务的信息
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent,
                    callerApp.uid, callerApp.processName, callingPackage);

            ...
			//如果绑定的时候带有BIND_AUTO_CREATE 标记，在服务没有创建的情况下，先创建服务
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                //bringUpServiceLocked 在前面启动服务的时候我们见过，这里为什么没有调用onStartCommand 因为 ServiceRecord#pendingStarts 长度为0 因此这个过程中只会调用onCreate
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }
 			...
			// b.intent.received 会在服务发布的时候，被置为true 
            if (s.app != null && b.intent.received) {
                // Service is already running, so we can immediately
                // publish the connection.
                try {
                    c.conn.connected(s.name, b.intent.binder, false);
                } catch (Exception e) {
                    Slog.w(TAG, "Failure sending service " + s.shortInstanceName
                            + " to connection " + c.conn.asBinder()
                            + " (in " + c.binding.client.processName + ")", e);
                }

                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                //针对同一个IntentFiler 匹配的 当绑定的进程数为1的时候 并且 IntentBindRecord #doRebind 为true
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) {
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }

           ...

        return 1;
    }
```

可以看到最后又会调用requestServiceBindingLocked（），它的最后一个参数会影响调用onBind()还是onRebind()

```java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
		...
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
                //发送消息开始进行anr监测
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                //将消息回调到 service所在的进程
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.getReportedProcState());
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            }
            ...
        }
        return true;
    }
```

最终会通过ActivityThread H handler 调用到 ActivityThread#handleBindService()

```java
 private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (DEBUG_SERVICE)
            Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
                        //调用的是onBind 
                        IBinder binder = s.onBind(data.intent);
                        //发布服务到ActivityService
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```

### ActivityManagerService发布服务的过程

ActivityManagerService#publishService最后会调用ActiveService#publishServiceLocked()

```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                    + " " + intent + ": " + service);
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                //注意这里是以 intent filter 作为key 意味着 当intent filter 不一样的时候又会执行onBind流程。
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    //保存onBind返回的binder代理对象
                    b.binder = service;
                    //重置相关标记
                    b.requested = true;
                    b.received = true;
                    ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                    for (int conni = connections.size() - 1; conni >= 0; conni--) {
                        ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Not publishing to: " + c);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Published intent: " + intent);
                                continue;
                            }
                            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                            try {
                                //回调服务已经连接 conn 是开始传递的LoadedApk#ServiceDispatcher#InnerConnection 的代理对象
                                c.conn.connected(r.name, service, false);
                            } catch (Exception e) {
                                Slog.w(TAG, "Failure sending service " + r.shortInstanceName
                                      + " to connection " + c.conn.asBinder()
                                      + " (in " + c.binding.client.processName + ")", e);
                            }
                        }
                    }
                }
				//解除anr监测
                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

### ServiceConnection连接回调过程

相对于其它的流程，连接回调相对较简单在绑定的时候根据传入的ServiceConnection 代理对象构造ConnectionRecord

```java
ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent,
                    callerApp.uid, callerApp.processName, callingPackage);
			//此时的binder 就是客户端传入的 InnerConnection 代理
            IBinder binder = connection.asBinder();
            s.addConnection(binder, c);
```

待服务发布后，再从ServiceRecord中取出对应的ConnectionRecord 进行回调即可。

但是这里并不是直接调用到我们自己的ServiceConnection而是在LoadedApk#ServiceDispatcher#InnerConnection#connected（）

而最后又会调用到ServiceDispatcher#connected（）

```java
 public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityExecutor != null) {
                mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
            } else if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {
                doConnected(name, service, dead);
            }
        }
```



## unBindService的过程

解除绑定最终会调用ActiveServices#removeConnectionLocked（）

```java
 void removeConnectionLocked(ConnectionRecord c, ProcessRecord skipApp,
            ActivityServiceConnectionsHolder skipAct) {
     ...
     //解绑服务
        s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
     ...
			//判断是否需要停止服务
            if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
                boolean hasAutoCreate = s.hasAutoCreateConnections();
                if (!hasAutoCreate) {
                    if (s.tracker != null) {
                        s.tracker.setBound(false, mAm.mProcessStats.getMemFactorLocked(),
                                SystemClock.uptimeMillis());
                    }
                }
                bringDownServiceIfNeededLocked(s, true, hasAutoCreate);
            }
        }
    }
```

在ActivityThread中会调用handleUnbindService

```java
private void handleUnbindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                boolean doRebind = s.onUnbind(data.intent);
                try {
                    if (doRebind) {//如果返回true 会在ActiveServices#unbindFinishedLocked 中重置相关标记，下次调用onRebind
                        ActivityManager.getService().unbindFinished(
                                data.token, data.intent, doRebind);
                    } else {
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to unbind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```



# anr的监测

根据前面的分析了解，Service anr 监测主要通过发送一个定时消息，如果在固定时间内没有移除对应消息就会触发anr

ActiveServices#bumpServiceExecutingLocked()

```java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        ...
        scheduleServiceTimeoutLocked(r.app);
            ...
        r.executeFg |= fg;
        r.executeNesting++;
        r.executingStart = now;
    }

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

发送的消息会在ActiveServices#serviceTimeout（）中进行处理

```java
void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;
        synchronized(mAm) {
            ...
            //这里会对anr进行处理
           mAm.mAnrHelper.appNotResponding(proc, anrMessage);
            ...
    }
```

解除anr监测：在ActiveServices#serviceDoneExecutingLocked（）

```java
 private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        ...
        //发送消息移除监听
        mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
     	...
    }
```



# 问题解答：

1. startService时为什么onCreate只调用一次，而onStartCommand却能够调用多次？

   onCreate的调用依据的是ServiceRecord#app 而在服务运行的时候这个对象会被赋值，此后在服务运行期间该对象都不为空，因此onCreate只会被调用一次。onStartCommand 调用的先决条件是ServiceRecord#pendingStarts的个数是否为0 当不为0 且服务在正常运行的时候，就能够调用onStartCommand（）。且pendingStarts有多少个就会调用多少次onStartCommand（）

2. bindService的过程中为什么onBind只会在第一次绑定的过程中调用？

   绑定服务的过程中，并不是onBind()方法只会调用一次，而是针对同一个IntentFilter onBind()只会调用一次。绑定服务有几个比较关键的数据结构

   - IntentBindRecord   以Intent#FilterComparison作为关键key,来保存绑定服务相关的信息

   - AppBindRecord  以进程为单位，持有绑定服务的相关信息

     ```java
     /**
      * An association between a service and one of its client applications.
      */
     final class AppBindRecord {
         final ServiceRecord service;    // The running service.
         final IntentBindRecord intent;  // The intent we are bound to.
         final ProcessRecord client;     // Who has started/bound the service.
     }
     ```

   - ConnectionRecord  绑定服务连接相关的信息

3. 为什么startService加bindService开启的服务需要同时调用stop和unbind来暂停？

   要停止服务需要将绑定数置位0且ServiceRecord#startRequested 为false  而stopService()只能将 startRequested 置为false，将绑定对象置为0 需要unBind来实现，因此二者都要调用。

4. 服务是如何检测anr的？

   anr 的检测是在每次通过IApplicationThread 交互的时候， 会发送一个延时消息到ActivitymanagerService#MainHandler 对象，如果到了固定的时间还没有解除这个消息，那么就会service 就会捕获anr。当程序交互正常进行，会在最后移除这个消息，此时会结束anr 监测。

5. 绑定服务之后，服务端是如何调用到客户端的？

   ServiceConnection 并不具备跨进程传输的能力，在绑定服务的过程中，系统会构造一个LoadedApk#ServiceDispacther#InnerConnection 对象进行跨进程传输，当连接信息发生改变的时候会回调到InnerConnection   而 InnerConnection  通过持有ServiceDispacther 而调用ServiceDispacther#connected  而ServiceDispacther持有我们传递进入的ServiceConnection 。从而代码会调用到我们自己的ServiceConnection 。需要注意的是ServiceDispacther 是以context为单位进行存储的，当通过activity或者更service 进行绑定，activity/service 结束时会自己解绑并清除对应的连接信息，我们即使不人为调用解绑操作也不会发生内存泄漏，但是这个并不是一个好习惯。

6. ServiceConnection#onServiceConnected在哪个线程执行？

   在api 30 的时候绑定服务可以提供一个Executor  handler 也就是说如果我们提供Excutor 那么就运行在Excutor 执行的线程池，否则看handler 是否为空，如果不为空，那么回调方法运行在handler所在的线程。否则运行在bind线程池。但是在绑定服务的时候如果我们不传递Excutor 系统默认会提供主线程handler。即默认情况下运行在主线程



如果还有一些关于服务的其它问题，欢迎留言，我们一起探究源码，解答相关的问题。

