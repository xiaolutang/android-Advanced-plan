# MediaSession的创建

```java

//MediaSession 的几个关键类
private final MediaSession.Token mSessionToken;
    private final MediaController mController;
    private final ISession mBinder;
    private final CallbackStub mCbStub;



public MediaSession(@NonNull Context context, @NonNull String tag, int userId) {
        if (context == null) {
            throw new IllegalArgumentException("context cannot be null.");
        }
        if (TextUtils.isEmpty(tag)) {
            throw new IllegalArgumentException("tag cannot be null or empty");
        }
        mMaxBitmapSize = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.config_mediaMetadataBitmapMaxSize);
        mCbStub = new CallbackStub(this);
        MediaSessionManager manager = (MediaSessionManager) context
                .getSystemService(Context.MEDIA_SESSION_SERVICE);
        try {
            //通过 MediaSessionManager 来创建一个跨进程的 Session
            mBinder = manager.createSession(mCbStub, tag, userId);
            mSessionToken = new Token(mBinder.getController());
            mController = new MediaController(context, mSessionToken);
        } catch (RemoteException e) {
            throw new RuntimeException("Remote error creating session.", e);
        }
    }
```

## MediaSessionManager创建ISession 对象

```java
private final ISessionManager mService;

public @NonNull ISession createSession(@NonNull MediaSession.CallbackStub cbStub,
            @NonNull String tag, int userId) throws RemoteException {
        return mService.createSession(mContext.getPackageName(), cbStub, tag, userId);
    }
```

mService 类型是ISessionManager 他的初始化在MediaSessionManager 的构造方法中  其中cbStub在构造方法中初始化，类型是CallbackStub

```java
public MediaSessionManager(Context context) {
        // Consider rewriting like DisplayManagerGlobal
        // Decide if we need context
        mContext = context;
        IBinder b = ServiceManager.getService(Context.MEDIA_SESSION_SERVICE);
        mService = ISessionManager.Stub.asInterface(b);
    }
```

在MediaSessionService 有这样发布了MEDIA_SESSION_SERVICE 对应的服务

```
publishBinderService(Context.MEDIA_SESSION_SERVICE, mSessionManagerImpl);
```

因此MediaSessionManager的createSession最终是调用的MediaSessionService内部类SessionManagerImpl 

### SessionManagerImpl 创建ISession

```java
@Override
        public ISession createSession(String packageName, ISessionCallback cb, String tag,
                int userId) throws RemoteException {
            final int pid = Binder.getCallingPid();
            final int uid = Binder.getCallingUid();
            final long token = Binder.clearCallingIdentity();
            try {
                enforcePackageName(packageName, uid);
                int resolvedUserId = ActivityManager.handleIncomingUser(pid, uid, userId,
                        false /* allowAll */, true /* requireFull */, "createSession", packageName);
                if (cb == null) {
                    throw new IllegalArgumentException("Controller callback cannot be null");
                }
                return createSessionInternal(pid, uid, resolvedUserId, packageName, cb, tag)
                        .getSessionBinder();
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
```

createSessionInternal会调用createSessionLocked

### MediaSessionService创建MediaSessionRecord

```java
private MediaSessionRecord createSessionLocked(int callerPid, int callerUid, int userId,
            String callerPackageName, ISessionCallback cb, String tag) {
        FullUserRecord user = getFullUserRecordLocked(userId);
        if (user == null) {
            Log.wtf(TAG, "Request from invalid user: " +  userId);
            throw new RuntimeException("Session request from invalid user.");
        }

        final MediaSessionRecord session = new MediaSessionRecord(callerPid, callerUid, userId,
                callerPackageName, cb, tag, this, mHandler.getLooper());
        try {
            cb.asBinder().linkToDeath(session, 0);
        } catch (RemoteException e) {
            throw new RuntimeException("Media Session owner died prematurely.", e);
        }

        user.mPriorityStack.addSession(session);
        mHandler.postSessionsChanged(userId);

        if (DEBUG) {
            Log.d(TAG, "Created session for " + callerPackageName + " with tag " + tag);
        }
        return session;
    }
```

可以看到最后会调用MediaSessionRecord#getSessionBinder

```java
public MediaSessionRecord(int ownerPid, int ownerUid, int userId, String ownerPackageName,
            ISessionCallback cb, String tag, MediaSessionService service, Looper handlerLooper) {
        mOwnerPid = ownerPid;
        mOwnerUid = ownerUid;
        mUserId = userId;
        mPackageName = ownerPackageName;
        mTag = tag;
        mController = new ControllerStub();
        mSession = new SessionStub();
        mSessionCb = new SessionCb(cb);
        mService = service;
        mContext = mService.getContext();
        mHandler = new MessageHandler(handlerLooper);
        mAudioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
        mAudioManagerInternal = LocalServices.getService(AudioManagerInternal.class);
        mAudioAttrs = new AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_MEDIA).build();
    }

    /**
     * Get the binder for the {@link MediaSession}.
     *
     * @return The session binder apps talk to.
     */
    public ISession getSessionBinder() {
        return mSession;
    }
```

这样MediaSession 持有跨进程的代理对象SessionStub 同时MediaSessionRecord 持有MeidaSession一路传递过来的CallbackStub对象，这样MediaSession 和 MediaSessionRecord就可以通过binder跨进程相互调用了。

## MediaSession#Token的创建

```java
 mSessionToken = new Token(mBinder.getController());
```

mBinder的实现类为MediaSessionRecord#SessionStub 

```java
@Override
public ISessionController getController() {
    return mController;
}
```

mController 是ControllerStub的实例，也就是说token 持有了ControllerStub 的跨进程代理。

## MediaController的创建

```java
public static final class Token implements Parcelable {

    private ISessionController mBinder;

    /**
     * @hide
     */
    public Token(ISessionController binder) {
        mBinder = binder;
    }

    ISessionController getBinder() {
        return mBinder;
    }
}
```

再创建MediaController的时候，直接将Token传递了过去

```java
    public MediaController(@NonNull Context context, @NonNull MediaSession.Token token) {
        //注意这个Binder是MediaSessionRecord#ControllerStub
        this(context, token.getBinder());
    }

public MediaController(Context context, ISessionController sessionBinder) {
        if (sessionBinder == null) {
            throw new IllegalArgumentException("Session token cannot be null");
        }
        if (context == null) {
            throw new IllegalArgumentException("Context cannot be null");
        }
        mSessionBinder = sessionBinder;
        mTransportControls = new TransportControls();
        mToken = new MediaSession.Token(sessionBinder);
        mContext = context;
    }
```

这样MediaController就具备了与MediaSessionRecord的交互能力。

