# 什么是ContentProvider?

ContentProvider是Android4大组件之一，我们平时使用的机会可能比较少。其底层通过Binder进行数据共享。如果我们要对第三方应用提供数据，可以考虑使用ContentProvider实现。

# 如何使用

## 与其他的ContentProvider通信。

要实现与其他的ContentProvider通信首先要查找到对应的ContentProvider进行匹配。android中ContenProvider借助ContentResolver通过Uri与其他的ContentProvider进行匹配通信。

### 认识Uri  来自：https://www.cnblogs.com/tgyf/p/4696288.html

> URI为系统中的每一个资源赋予一个名字，比方说通话记录。每一个ContentProvider都拥有一个公共的URI，用于表示ContentProvider所提供的数据。 Android所提供的ContentProvider都位于android.provider包中， 可以将URI分为A、B、C、D 4个部分来理解。如对于content://com.wang.provider.myprovider/tablename/id：
>
> 　　a、标准前缀——content://，用来说明一个Content Provider控制这些数据；
>
> 　　b、URI的标识——com.wang.provider.myprovider，用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。对于第三方应用程序，为了保证URI标识的唯一性，它必须是一个完整的、小写的类名。这个标识在元素的authorities属性中说明，一般是定义该ContentProvider的包.类的名称；
>
> 　　c、路径——tablename，通俗的讲就是你要操作的数据库中表的名字，或者你也可以自己定义，记得在使用的时候保持一致就可以了；
>
> 　　d、记录ID——id，如果URI中包含表示需要获取的记录的ID，则返回该id对应的数据，如果没有ID，就表示返回全部；
>
> 　　对于第三部分路径（path）做进一步的解释，用来表示要操作的数据，构建时应根据实际项目需求而定。如：
>
>    a、操作tablename表中id为11的记录，构建路径：/tablename/11；
>
>    b、操作tablename表中id为11的记录的name字段：tablename/11/name；
>
>    c、操作tablename表中的所有记录：/tablename；
>
>    d、操作来自文件、xml或网络等其他存储方式的数据，如要操作xml文件中tablename节点下name字段：/ tablename/name；
>
>    e、若需要将一个字符串转换成Uri，可以使用Uri类中的parse()方法，如：
>
> ```java
> Uri uri = Uri.parse("content://com.wang.provider.myprovider/tablename")；
> ```

### 最简单的查询ContentProvider

1. 通过Context获取ContentResolver
2. 调用它的query方法

```java
ContentResolver resolver = getContentResolver();
Cursor cursor = resolver.query(Uri.parse(""),null,null,null,null);
if(cursor != null){
    while (cursor.moveToNext()){
        Log.d("tag","query result "+cursor.getColumnNames());
    }
    cursor.close();
}
```

## 实现自己的ContentProvider

实现自的ContentProvider需要继承Android系统的ContentProvider然后实现下面的几个方法。

- onCreate()
- query()
- getType()
- insert()
- delete()
- update()

需要注意的是除了onCreate()其他的方法都运行在binder线程池。

然后在Manifest中声明对应的contentProvider即可。

**contentProvider实现**

```kotlin
class DataContentProvider : ContentProvider() {
    private val tag = "DataContentProvider"

    private var dbHelper:DBHelper? = null

    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        return uri
    }

    override fun query(uri: Uri, projection: Array<String>?, selection: String?, selectionArgs: Array<String>?, sortOrder: String?): Cursor? {
        val cursor  = dbHelper?.readableDatabase?.query(DBHelper.USER_TABLE_NAME,projection,null,selectionArgs,null,null,sortOrder)
        Log.d(tag,"call query cursor is $cursor")
        return cursor
    }

    override fun onCreate(): Boolean {
        dbHelper = DBHelper(this.context)
        val db = dbHelper?.writableDatabase

        db?.execSQL("delete from ${DBHelper.USER_TABLE_NAME}");
        db?.execSQL("insert into ${DBHelper.USER_TABLE_NAME} values(1,'XW');")
        db?.execSQL("insert into ${DBHelper.USER_TABLE_NAME} values(2,'XZ');")
        return true
    }

    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<String>?): Int {
        return 0
    }

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int {
        return 0
    }

    override fun getType(uri: Uri): String? {
        return null
    }
}
```

```kotlin
class DBHelper  //数据库版本号
(context: Context?) : SQLiteOpenHelper(context, DATABASE_NAME, null, DATABASE_VERSION) {
    override fun onCreate(db: SQLiteDatabase) { // 创建两个表格:用户表 和职业表
        db.execSQL("CREATE TABLE IF NOT EXISTS $USER_TABLE_NAME(_id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT)")
    }

    override fun onUpgrade(db: SQLiteDatabase?, oldVersion: Int, newVersion: Int) {}

    companion object {
        // 数据库名
        private const val DATABASE_NAME = "demo_provider.db"
        // 表名
        const val USER_TABLE_NAME = "user"
        private const val DATABASE_VERSION = 1
    }
}
```

Manifest注册如下：

```xml
   <provider
            android:name=".contentprovider.DataContentProvider"
            android:authorities="com.txl.demo.content.provider" />
```

这样一个及其简单的ContentProvider就实现了。里面只实现了查询功能。

## 监听ContentProvider的数据变化 

当ContentProvider数据发生改变的时候,可以通过ContentResolver的notifyChange（）通知监听者数据发生改变。而外部需要通过ContentResolver注册监听才能接收到数据变化通知。

# 工作流程

要理解这个工作流程需要对Android的Binder通信机制有较好的理解。可以参考 

[Binder学习笔记]: https://blog.csdn.net/qq_35561554/article/details/102810365

我们都知道ContentProvider通过binder向其他组件或者应用程序提供数据。

> 当ContentProvider所在的进程启动的时候，ContentProvider会同时启动并被发布到AMS中，需要注意的是ContentProvider的onCreate方法会先于Application的OnCreate调用。

应用程序启动的时候会调用ActivityThread#main方法。在这里会创建ActivityThread实例并初始化主线程的消息队列（初始化主线程的消息Looper）。并在ActivityThread#attach方法中远程调用AMS的attachApplication方法。而AMS又会远程调用ActivityThread#ApplicationThread#bindApplication。在bindApplication方法中通过handler切换到ActivityThread#handleBindApplication这里会创建Application和ContentProvider。

我们在handleBindApplication找到了 下面的一段代码：

```java
 app = data.info.makeApplication(data.restrictedBackupMode, null);// Propagate autofill compat state
            app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);

            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    //创建ContentProvider
                    installContentProviders(app, data.providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }
            try {
                //调用Application的OnCreate
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                      "Unable to create application " + app.getClass().getName()
                      + ": " + e.toString(), e);
                }
            }
```

上面的代码可以证明前面说的ContentProvider创建在Application#onCreate调用之前。

## ContentProvider的query过程

为什么分析query过程？

1.query是Content最常见的一个使用流程，具有代表性。

2.ContentProvider跨进程通信返回了一个未经过Parcelable序列化的Cursor。这让人不得不好奇这个过程经历了什么。

我们通过Context#getContentResolver获取 ContentResolver。Conetx获取到的ContentResolver是ApplicationContentResolver对象。

ApplicationContentResolver的query方法在它的父类中实现ContentResolver中实现，在query中首先会获取 一个IContentProvider对象，不管是通过 acquireUnstableProvider 方法还是通过acquireProvider（）方法其本质最终都是通过调研ActivityThread#acquireProvider方法来实现

ActivityThread#acquireProvider

```java
public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        // There is a possible race here.  Another thread may try to acquire
        // the same provider at the same time.  When this happens, we want to ensure
        // that the first one wins.
        // Note that we cannot hold the lock while acquiring and installing the
        // provider since it might take a long time to run and it could also potentially
        // be re-entrant in the case where the provider is in the same process.
        ContentProviderHolder holder = null;
        try {
            synchronized (getGetProviderLock(auth, userId)) {
                holder = ActivityManager.getService().getContentProvider(
                        getApplicationThread(), auth, userId, stable);
            }
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```

上面的代码逻辑是：首先查询是否存在对应的ContentProvider,如果不存在则通过ActivityManagerServer获取对应的ContentProviderHolder，在ActivityManager获取ContentProviderHolder的过程中会判断contentProvider所在的进程是否存在如果不存在的话会创建对应的进程并启动ContentProvider。最后调用installProvider来获取IContentProvider对象。

```java
private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {
            if (DEBUG_PROVIDER || noisy) {
                Slog.d(TAG, "Loading provider " + info.authority + ": "
                        + info.name);
            }
            Context c = null;
            ApplicationInfo ai = info.applicationInfo;
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null &&
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try {
                    c = context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }
            if (c == null) {
                Slog.w(TAG, "Unable to get context for package " +
                      ai.packageName +
                      " while loading content provider " +
                      info.name);
                return null;
            }

            if (info.splitName != null) {
                try {
                    c = c.createContextForSplit(info.splitName);
                } catch (NameNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }

            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
                LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
                if (packageInfo == null) {
                    // System startup case.
                    packageInfo = getSystemContext().mPackageInfo;
                }
                localProvider = packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                provider = localProvider.getIContentProvider();
                if (provider == null) {
                    Slog.e(TAG, "Failed to instantiate class " +
                          info.name + " from sourceDir " +
                          info.applicationInfo.sourceDir);
                    return null;
                }
                if (DEBUG_PROVIDER) Slog.v(
                    TAG, "Instantiating local provider " + info.name);
                // XXX Need to create the correct context for this provider.
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            "Unable to get provider " + info.name
                            + ": " + e.toString(), e);
                }
                return null;
            }
        } else {
            provider = holder.provider;
            if (DEBUG_PROVIDER) Slog.v(TAG, "Installing external provider " + info.authority + ": "
                    + info.name);
        }

        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            if (DEBUG_PROVIDER) Slog.v(TAG, "Checking to add " + provider
                    + " / " + info.name);
            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, "
                                + "using existing local provider");
                    }
                    provider = pr.mProvider;
                } else {
                    holder = new ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder = pr.mHolder;
            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                if (prc != null) {
                    if (DEBUG_PROVIDER) {
                        Slog.v(TAG, "installProvider: lost the race, updating ref count");
                    }
                    // We need to transfer our new reference to the existing
                    // ref count, releasing the old one...  but only if
                    // release is needed (that is, it is not running in the
                    // system process).
                    if (!noReleaseNeeded) {
                        incProviderRefLocked(prc, stable);
                        try {
                            ActivityManager.getService().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            //do nothing content provider object is dead any way
                        }
                    }
                } else {
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    if (noReleaseNeeded) {
                        prc = new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc = stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder = prc.holder;
            }
        }
        return retHolder;
    }
```

整个installProvider方法的代码比较长，核心思想是，如果传入的holder为空或者它的provider为空，那么会执行本地创建逻辑，调用创建的ContentProvider#getIContentProvider()获取IContentProvider对象。然后放入对应的Holder。如果传入的holder不为空，直接获取holder中的provider。也就是说我们可以简单理解为获取到的IContentProvider为getIContentProvider方法返回的对象。

```java
public IContentProvider getIContentProvider() {
        return mTransport;
    }
```

可以看到ContentProvider#getIContentProvider返回了mTransport，mTransport对应的类为ContentProvider#Transport，Transport继承了ContentProviderNative，而ContentProviderNative又继承了BInder。这样我们就明白了Transport通过继承Binder的方式实现了跨进程传输。我们知道Binder通信跨进程进行方法调用时通过onTransact才能处理到对应的方法。我们直接看到ContentProviderNative#onTransact。

因为onTransact方法实现过长，这里我们只关心query的实现部分。

```java
case QUERY_TRANSACTION:
                {
                    data.enforceInterface(IContentProvider.descriptor);

                    String callingPkg = data.readString();
                    Uri url = Uri.CREATOR.createFromParcel(data);

                    // String[] projection
                    int num = data.readInt();
                    String[] projection = null;
                    if (num > 0) {
                        projection = new String[num];
                        for (int i = 0; i < num; i++) {
                            projection[i] = data.readString();
                        }
                    }

                    Bundle queryArgs = data.readBundle();
                    IContentObserver observer = IContentObserver.Stub.asInterface(
                            data.readStrongBinder());
                    ICancellationSignal cancellationSignal = ICancellationSignal.Stub.asInterface(
                            data.readStrongBinder());

                    Cursor cursor = query(callingPkg, url, projection, queryArgs, cancellationSignal);
                    if (cursor != null) {
                        CursorToBulkCursorAdaptor adaptor = null;

                        try {
                            adaptor = new CursorToBulkCursorAdaptor(cursor, observer,
                                    getProviderName());
                            cursor = null;

                            BulkCursorDescriptor d = adaptor.getBulkCursorDescriptor();
                            adaptor = null;

                            reply.writeNoException();
                            reply.writeInt(1);
                            d.writeToParcel(reply, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                        } finally {
                            // Close cursor if an exception was thrown while constructing the adaptor.
                            if (adaptor != null) {
                                adaptor.close();
                            }
                            if (cursor != null) {
                                cursor.close();
                            }
                        }
                    } else {
                        reply.writeNoException();
                        reply.writeInt(0);
                    }

                    return true;
                }
```

在这个主要涉及到这两个类CursorToBulkCursorAdaptor，BulkCursorDescriptor。BulkCursorDescriptor实现了Parcelable可以跨进程进行传输，CursorToBulkCursorAdaptor间接继承了Binder页具备跨进程传输能力。在跨进程返回数据的时将BulkCursorDescriptor写入到返回数据中。在ContentProviderProxy中进行反序列化得到CursorToBulkCursorAdaptor类。这样整个查询过程就清晰了。

## ContentProvider的发布过程

对于ContentProvider#getIContentProvider补充说明，前面我们直接简单理解成返回了mTransport这个是有点问题的。其实在哪个位置获取到的IContentProvider对象是ContentProviderProxy。我们来看看原因。

在应用程序启动的时候会启动ContentProvider，并将对用的ContentProvider发布到ActivityManagerServer。

```java
private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            //将ContentProvider相关信心发布到ActivityManagerService
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

可以看到最终将results发布到了ActivityManagerService。而这个result列表里面装的是ContentProviderHolder。它实现了Parcelable。我们知道跨进程传输需要涉及到序列化与反序列化。

```java
public class ContentProviderHolder implements Parcelable {
    public final ProviderInfo info;
    public IContentProvider provider;
    public IBinder connection;
    public boolean noReleaseNeeded;

    public ContentProviderHolder(ProviderInfo _info) {
        info = _info;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        info.writeToParcel(dest, 0);
        if (provider != null) {
            dest.writeStrongBinder(provider.asBinder());
        } else {
            dest.writeStrongBinder(null);
        }
        dest.writeStrongBinder(connection);
        dest.writeInt(noReleaseNeeded ? 1 : 0);
    }

    public static final Parcelable.Creator<ContentProviderHolder> CREATOR
            = new Parcelable.Creator<ContentProviderHolder>() {
        @Override
        public ContentProviderHolder createFromParcel(Parcel source) {
            return new ContentProviderHolder(source);
        }

        @Override
        public ContentProviderHolder[] newArray(int size) {
            return new ContentProviderHolder[size];
        }
    };

    private ContentProviderHolder(Parcel source) {
        info = ProviderInfo.CREATOR.createFromParcel(source);
        provider = ContentProviderNative.asInterface(
                source.readStrongBinder());
        connection = source.readStrongBinder();
        noReleaseNeeded = source.readInt() != 0;
    }
}
```

可以看到 反序列化的时候通过ContentProviderNative#asInterface得到了 ContentProviderProxy的实例对象。因此在跨进程是我们 getIContentProvider时获取到的是ContentProviderProxy实例对象。这是一个典型的Binder通信过程。

# autosize中ContentProvider

AutoSize是一款优秀的基于今日头条方案的屏幕适配框架，它的使用非常简单。引入框架，然后在manifest中声明对应设计图的宽高即可使用。侵入性非常低。那么开启应用即可使用的呢？答案是利用ContentProvider会随应用程序启动而启动的特性。然后在ContentProvider中做自己的初始化操作。这个是一个非常好的实现思路。我们不仅可以用ContentProvider提供数据，也可以利用它的特性初始化自己的特定逻辑。

# 总结:

1. 如何通过ContentProvider查询数据

   通过ContentResolver 进行uri匹配

2. 如何实现自己的ContentProvider

   继承ContentProvider,实现对应的方法。在manifest中声明

3. ContentResolver如何返回Cursor对象

   在跨进程的情况下返回的是CursorToBulkCursorAdaptor对象，其实质是借助Binder的跨进程传输能力，在ContentProvider进程中序列化，在调用程序中反序列化。