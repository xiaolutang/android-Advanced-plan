# 概述

什么是MediaProvider?  MediaProvider是Android多媒体系统中极其重要的一员，他是一个独立的应用程序。MediaProvider存在的作用是对文件进行扫描存储到数据库，这样后续图片，音频、视频等应用就可以直接查询展示相关的信息，而不必每次启动应用重新进行扫描处理。MediaProvider应用程序内部主要包含下面的几个部分：

**MediaProvider：**Android四大组件之一ContentProvider 。第三方应用可以通过ContentProvider标准接口来查询多媒体相关的信息。

**MediaScanner:**文件扫描工具，包含java 和C++两个部分

**MediaScannerReceiver/MediaScannerService：**用于接收扫描信息，并发起扫描。



# 扫描流程

MediaScannerReceiver通过静态注册的方式接收开机完成、U盘插拔等广播，每次U盘插拔会接收到对应的广播消息。

## MediaScannerReceiver对于U盘插入的处理。

```java
@Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        final Uri uri = intent.getData();
        StorageVolume storage = (StorageVolume) intent.getParcelableExtra(
                StorageVolume.EXTRA_STORAGE_VOLUME);
        int storageId = (storage != null) ?
                ScanUtil.getStorageIdForUuid(storage.getUuid()) : StorageVolume.STORAGE_ID_INVALID;

       ....
                if (Intent.ACTION_MEDIA_MOUNTED.equals(action)) {
                   if (storage != null) {
                       //是否支持模拟器扫描
                        boolean supportEmulated = context.getResources()
                                .getBoolean(R.bool.support_emulated_storage);
                       //如果当前设备是私有设备或者是不可移除的设备不对其进行设备表的维护
                        if (storage.isPrimary() || !storage.isRemovable()) {
                            if (!supportEmulated) {//模拟器的内部存储如果不允许扫描，直接退出
                                Log.d(TAG, "Not support scanning emulated storage and return.");
                                return;
                            }
                        } else {
                            //恢复设备表信息
                            StorageCheckUtil.checkAndUpdateStorageTable(context, storage, path);
                        }
                       	//刷新内存中维护的存储设备列表
                        ScanUtil.refreshStorageIdList(Integer.toString(storageId), true);
                       	//发起扫描
                        scan(context, MediaProvider.EXTERNAL_VOLUME, path, storageId);
                    }
                } 
        ....
            }
        }
    }
```

MediaScannerReceiver对于U盘的插入有两个比较重要的操作

1. 维护U盘设备列表
2. 发起扫描操作。

### U盘设备维护

U盘设备的维护操作通过StorageCheckUtil#checkAndUpdateStorageTable来完成，内部的主要逻辑是将当前数据库中维护的设备列表与StorageManager查询到的设备进行一一对比，如果查询的的设备未被使用，且上次的使用时间间隔超过180天（默认的，可以进行配置）那么将对应的信息进行删除。同时设备列表最多维护三个设备信息，当超过3个的时候会将距离当前使用时间最久的那个设备信息移除。

**需要注意的是：**在源码中虽然对设备信息进行了移除，但是扫描入库的文件没有在这个时候进行移除。

### 发起扫描

扫描由MediaScannerReceiver#scan发起，其本质上就是通过启动MediaScannerService服务来发起扫描。

```java
private void scan(Context context, String volume, String path, int storageId) {
    Bundle args = new Bundle();
    args.putString(ScanUtil.STORAGE_VOLUME, volume);
    args.putString(ScanUtil.SCAN_PATH, path);
    args.putInt(ScanUtil.SCAN_PATH_ID, storageId);
    context.startService(
            new Intent(context, MediaScannerService.class).putExtras(args));
}  
```

### 结束扫描

当接收到U盘拔出的时候MediaScannerReceiver会调用ScanUtil#abortScan结束本次扫描工作

```java
public static void abortScan(Context context, String volume, String path, int storageId) {
    Bundle args = new Bundle();
    args.putString(STORAGE_VOLUME, volume);
    args.putString(SCAN_PATH, path);
    args.putString(ABORT_ACTION, ABORT_SCAN_ACTION);
    args.putInt(SCAN_PATH_ID, storageId);
    context.startService(
            new Intent(context, MediaScannerService.class).putExtras(args));
}
```

## MediaScannerService的工作

MediaScannerService继承自Service实现了Runnable接口，同时内部封装了Hander Thread线程来进行文件扫描。大部分情况下MediaScannerService通过startService进行工作，同时也支持绑定服务。

在MediaScannerService#onStartCommand主要的工作是，解析startService传递过来的参数，如果是结束扫描那么就终止对应MediaScanner的扫描过程。否则通过Handler转发到子线程发起扫描工作。

### 终止扫描过程

```java
public static void abortScan(Context context, MediaScanner scanner, String path) {
    if (scanner != null) {
        //通知底层结束扫描
        scanner.setAbortScanPath(path);
        //发送扫描异常广播
        ScanUtil.sendExceptionBroadcast(context, ScanUtil.SCAN_ABORT,
                "Current scan abort! path = " + path);
    }
}
```

### 扫描的过程

扫描有许多的逻辑判断，可以扫描指定路径，也可以扫描内部存储。这里我们着重与关注外部存储的扫描过程。

外部存储的扫描会先判断当前是否有足够多的存储空间，如果数据库的大小超过默认配置的大小，那么会尝试删除没有使用的多媒体信息。

在空间充足的情况下会调用scan扫描。

```java
private void scan(String[] directories, String volumeName) {
    Uri uri = Uri.parse("file://" + directories[0]);
    // don't sleep while scanning
    mWakeLock.acquire();

    try {
        ContentValues values = new ContentValues();
        values.put(MediaStore.MEDIA_SCANNER_VOLUME, volumeName);
        //查询MediaProvider是否有相关的数据库  如果有的话更新扫描开始时间
        Uri scanUri = getContentResolver().insert(MediaStore.getMediaScannerUri(), values);
		//发送开始扫描的消息
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_STARTED, uri));

        try {
            //如果是外部存储
            if (volumeName.equals(MediaProvider.EXTERNAL_VOLUME)) {
                //查找或创建对应的databaserHelper
                openDatabase(volumeName);
            }

            try (MediaScanner scanner = new MediaScanner(this, volumeName)) {
                // Put the current scan process and scan path into the Pair.
                // If it is external volume, only one scan path is stored in directories.
                int scope = getBaseContext().getResources().getInteger(R.integer.metadata_scope);
				//根据这个scope 决定获取多媒体信息的那些内容
                scanner.setMetadataScope(scope);

                if (volumeName.equals(MediaProvider.EXTERNAL_VOLUME)
                        && !directories[0].equals(ScanUtil.EXTERNAL_EMULATED)) {
                    mCurrentMediaScannerAndPath = new Pair<>(directories[0], scanner);
                }
                //发起扫描
                scanner.scanDirectories(directories);
            }
        } catch (Exception e) {
            Log.e(TAG, "exception in MediaScanner.scan()", e);
        }

        getContentResolver().delete(scanUri, null, null);

    } finally {
        // If one finish scanning,let mCurrentMediaScannerAndPath be equal to null.
        mCurrentMediaScannerAndPath = null;
        //扫描结束 发送广播
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_FINISHED, uri));
        mWakeLock.release();
    }
}
```

## MediaScanner的扫描过程

MediaScanner的扫描工作在MediaScanner#scanDirectories发起，scanDirectories主要有三个工作，

1. 调用prescan做一些扫描前的准备工作，主要是通过读取当前数据库中的文件，与U盘中的文件进行对比，将不存在的文件从数据库中删除。**这个位置要验证：因为没有进行U盘区分，如果上一个u盘内容非常多，会不会影响下一个U盘的同步呢？**
2. 在scanDirectories会遍历所有待扫码的文件夹路径，然后通过jni调用processDirectory通知native层对当前的文件夹进行扫描。
3. 调用postscan处理扫描后续的工作

### C++层深度优先遍历文件夹

文件夹扫描是通过processDirectory通知native来完成，它的执行代码位于TsMediaScanner.cpp的TsMediaScanner#processDirectory而processDirectory内部又会通过doProcessDirectory来完成扫描过程。

```c++
MediaScanResult TsMediaScanner::doProcessDirectory(
        char *path, int pathRemaining, MediaScannerClient &client, bool noMedia, int depth) {
    // place to copy file or directory name
    char* fileSpot = path + strlen(path);
    struct dirent* entry;
    ALOGV("doProcessDirectory: path=[%s]", path);
	//判断当前路径是不是需要跳过扫描
    if (shouldSkipDirectory(path)) {
        ALOGD("Skipping: %s", path);
        return MEDIA_SCAN_RESULT_OK;
    }

    // Treat all files as non-media in directories that contain a  ".nomedia" file
    //剩下的地址空间大于8  因为可能会追加 .nomedia
    if (pathRemaining >= 8 /* strlen(".nomedia") */ ) {
        //路径字符串后追加.nomedia
        //如果该文件能找到.nomedia标识，直接忽略
        strcpy(fileSpot, ".nomedia");
        if (access(path, F_OK) == 0) {
            ALOGV("found .nomedia, setting noMedia flag");
            noMedia = true;
        }

        // restore path
        fileSpot[0] = 0;
    }

    if (noMedia) {
        ALOGD("Ignore nomedia item: [%s]", path);
        return MEDIA_SCAN_RESULT_OK;
    }

    //打开当前文件夹
    DIR* dir = opendir(path);
    if (!dir) {
        ALOGW("Error opening directory '%s', skipping: %s.", path, strerror(errno));
        return MEDIA_SCAN_RESULT_SKIPPED;
    }

    MediaScanResult result = MEDIA_SCAN_RESULT_OK;
    std::list<EntryDirent*> entryListFolder;
    while ((entry = readdir(dir))) {
        // If the aborted directory is the currently a scanned directory, abort scan.
        //判断当前扫描是不是被终止了
        if (mAbortScanPath != NULL && (strcmp(path, mAbortScanPath) == 0
                        || (strncmp(path, mAbortScanPath, strlen(mAbortScanPath)) == 0
                        && path[strlen(mAbortScanPath)] == '/'))) {
            ALOGW("Aborting directory '%s'", mAbortScanPath);
            result = MEDIA_SCAN_RESULT_ERROR;
            break;
        }
        const char* name = entry->d_name;
        int type = entry->d_type;
        // ALOGV("doProcessDirectory: while: name=[%s] type=[%d]", name, type);
        //如果是文件 调用doProcessDirectoryEntry先进行处理
        if (type == DT_REG) {  // Deal with files first, then folders.
            EntryDirent newEntry;
            newEntry.d_type = type;
            newEntry.d_name = strdup(name);
            if (doProcessDirectoryEntry(path, pathRemaining, client, noMedia, &newEntry, fileSpot, depth)
                    == MEDIA_SCAN_RESULT_ERROR) {
                ALOGW("Scan error! Break out from file-entry '%s'", newEntry.d_name);
                free(newEntry.d_name);
                result = MEDIA_SCAN_RESULT_ERROR;
                break;
            } else {
                free(newEntry.d_name);
            }
        } else {//如果是文件夹，将其放入集合
            EntryDirent* newEntry = new EntryDirent();
            newEntry->d_type = type;
            newEntry->d_name = strdup(name);
            if (1 == depth) {  // Root path.
                if (strcmp(name, "Android") == 0) {
                    entryListFolder.push_back(newEntry);
                } else {
                    entryListFolder.push_front(newEntry);
                }
            } else {
                entryListFolder.push_back(newEntry);
            }
        }
    }
    //遍历集合，扫描文件夹
    if (!entryListFolder.empty()) {
        for (std::list<EntryDirent*>::iterator it = entryListFolder.begin(); it != entryListFolder.end(); ++it) {
            EntryDirent* newEntry = *it;
            // If the aborted directory is the currently a scanned directory, abort scan.
            if (mAbortScanPath != NULL && (strcmp(path, mAbortScanPath) == 0
                            || (strncmp(path, mAbortScanPath, strlen(mAbortScanPath)) == 0
                            && path[strlen(mAbortScanPath)] == '/'))) {
                ALOGW("Aborting directory '%s'", mAbortScanPath);
                result = MEDIA_SCAN_RESULT_ERROR;
                break;
            }
            if (doProcessDirectoryEntry(path, pathRemaining, client, noMedia, newEntry, fileSpot, depth)
                    == MEDIA_SCAN_RESULT_ERROR) {
                ALOGW("Scan error! Break out from non-file-entry '%s' in the folder '%s'", newEntry->d_name, path);
                result = MEDIA_SCAN_RESULT_ERROR;
                break;
            }
        }
        for (std::list<EntryDirent*>::iterator it = entryListFolder.begin(); it != entryListFolder.end(); ++it) {
            EntryDirent* newEntry = *it;
            free(newEntry->d_name);
            delete newEntry;
        }
        entryListFolder.clear();
    }
    closedir(dir);
    return result;
}
```

可以看到doProcessDirectory只是处理扫描顺序，执行扫描的在doProcessDirectoryEntry

```c++
MediaScanResult TsMediaScanner::doProcessDirectoryEntry(
        char *path, int pathRemaining, MediaScannerClient &client, bool noMedia,
        EntryDirent* entry, char* fileSpot, int depth) {
    struct stat statbuf;
    const char* name = entry->d_name;
    ALOGV("doProcessDirectoryEntry: path=[%s] fileSpot=[%s] name=[%s] type=%d", path, fileSpot, name, entry->d_type);

    // ignore "." and ".."
    if (name[0] == '.' && (name[1] == 0 || (name[1] == '.' && name[2] == 0))) {
        return MEDIA_SCAN_RESULT_SKIPPED;
    }

    int nameLength = strlen(name);
    if (nameLength + 1 > pathRemaining) {
        // path too long!
        return MEDIA_SCAN_RESULT_SKIPPED;
    }
    strcpy(fileSpot, name);

    int type = entry->d_type;
    //如果当前文件类型位置，再读一次文件信息，尝试从中获取文件类型
    if (type == DT_UNKNOWN) {
        // If the type is unknown, stat() the file instead.
        // This is sometimes necessary when accessing NFS mounted filesystems, but
        // could be needed in other cases well.
        if (stat(path, &statbuf) == 0) {
            if (S_ISREG(statbuf.st_mode)) {
                type = DT_REG;
                ALOGV("stat() success for '%s', treat as file", path);
            } else if (S_ISDIR(statbuf.st_mode)) {
                type = DT_DIR;
                ALOGV("stat() success for '%s', treat as folder", path);
            }
        } else {
            ALOGD("stat() failed for %s: %s", path, strerror(errno) );
        }
    }
    //文件夹
    if (type == DT_DIR) {
        // If the mMaxScanDepth is 0,represent support the full scan
        if (mMaxScanDepth > 0 && depth > mMaxScanDepth) {
            ALOGV("%s overdue max scan depth", path);
            return MEDIA_SCAN_RESULT_OK;
        }
        bool childNoMedia = noMedia;
        // set noMedia flag on directories with a name that starts with '.'
        // for example, the Mac ".Trashes" directory
        if (name[0] == '.')
            childNoMedia = true;

        if (childNoMedia) {
            ALOGD("Ignore nomedia item: path=[%s] fileSpot=[%s] name=[%s]", path, fileSpot, name);
            return MEDIA_SCAN_RESULT_OK;
        }

        // report the directory to the client
        if (stat(path, &statbuf) == 0) {
            //通知java层读取文件夹
            status_t status = client.scanFile(path, statbuf.st_mtime, 0,
                    true /*isDirectory*/, childNoMedia);
            if (status) {
                return MEDIA_SCAN_RESULT_ERROR;
            }
        }

        // and now process its contents
        strcat(fileSpot, "/");
        //调用doProcessDirectory 遍历当前文件夹
        MediaScanResult result = doProcessDirectory(path, pathRemaining - nameLength - 1,
                client, childNoMedia, depth + 1);
        if (result == MEDIA_SCAN_RESULT_ERROR) {
            return MEDIA_SCAN_RESULT_ERROR;
        }
    } else if (type == DT_REG) {//文件
        if (stat(path, &statbuf) == 0) {
            //通知java层读取文件
            status_t status = client.scanFile(path, statbuf.st_mtime, statbuf.st_size,
                    false /*isDirectory*/, noMedia);
            if (status) {
                return MEDIA_SCAN_RESULT_ERROR;
            }
        }
    }

    return MEDIA_SCAN_RESULT_OK;
}
```

这个里面的client  是C++层的对象MyMediaScannerClient，它的内部通过JNI反向调用java层  MediaScanner#MyMediaScannerClient

 MediaScanner#MyMediaScannerClient#scanFile

```java
public void scanFile(String path, long lastModified, long fileSize,
        boolean isDirectory, boolean noMedia) {
    // This is the callback funtion from native codes.
    // Log.v(TAG, "scanFile: "+path);
    doScanFile(path, null, lastModified, fileSize, isDirectory, false, noMedia);
}
```

### doScanFile的过程

```java
public Uri doScanFile(String path, String mimeType, long lastModified,
                long fileSize, boolean isDirectory, boolean scanAlways, boolean noMedia) {
            Uri result = null;

            // mScanFileCount is used to calculate the scan progress. If the scan path is a file,
            // ++mScanFileCount.
    //更新当前扫描进度
            if (mNeedProgressUpdate && new File(path).isFile()) {
                ++mScanFileCount;
            }
//            long t1 = System.currentTimeMillis();
            try {
                //构建文件结构，beginFile 会尝试先从数据库查询，如果不存在则创建新的
                FileEntry entry = beginFile(path, mimeType, lastModified,
                        fileSize, isDirectory, noMedia);

                if (entry == null) {
                    return null;
                }
                //If it is not a file of type audio, video or image,skip the current scan and
                // continue processing the next file.
                //如果不是  音频 、视频、 图片、音频扩展文件，跳过当前文件扫描
                if (!(MediaFile.isAudioFileType(mFileType)
                        || MediaFile.isVideoFileType(mFileType)
                        || MediaFile.isImageFileType(mFileType)
                        || MediaExtensionFile.isAudioExtensionFileType(mFileType))) {
                    return null;
                }
                // if this file was just inserted via mtp, set the rowid to zero
                // (even though it already exists in the database), to trigger
                // the correct code path for updating its entry
                if (mMtpObjectHandle != 0) {
                    entry.mRowId = 0;
                }

                if (entry.mPath != null) {
                    if (((!mDefaultNotificationSet &&
                                doesPathHaveFilename(entry.mPath, mDefaultNotificationFilename))
                        || (!mDefaultRingtoneSet &&
                                doesPathHaveFilename(entry.mPath, mDefaultRingtoneFilename))
                        || (!mDefaultAlarmSet &&
                                doesPathHaveFilename(entry.mPath, mDefaultAlarmAlertFilename)))) {
                        Log.w(TAG, "forcing rescan of " + entry.mPath +
                                "since ringtone setting didn't finish");
                        scanAlways = true;
                    } else if (isSystemSoundWithMetadata(entry.mPath)
                            && !Build.FINGERPRINT.equals(sLastInternalScanFingerprint)) {
                        // file is located on the system partition where the date cannot be trusted:
                        // rescan if the build fingerprint has changed since the last scan.
                        Log.i(TAG, "forcing rescan of " + entry.mPath
                                + " since build fingerprint changed");
                        scanAlways = true;
                    }
                }

                // rescan for metadata if file was modified since last scan
                if (entry.mLastModifiedChanged || scanAlways) {
                    if (noMedia) {
                        result = endFile(entry, false, false, false, false, false);
                    } else {
                        boolean isaudio = MediaFile.isAudioFileType(mFileType)
                                ||MediaExtensionFile.isAudioExtensionFileType(mFileType);
                        boolean isvideo = MediaFile.isVideoFileType(mFileType);
                        boolean isimage = MediaFile.isImageFileType(mFileType);

                        if (isaudio || isvideo || isimage) {
                            path = Environment.maybeTranslateEmulatedPathToInternal(new File(path))
                                    .getAbsolutePath();
                        }

                        // we only extract metadata for audio and video files
                        if (isaudio || isvideo) {//解析音视频文件
                            //int scope = mContext.getResources().getInteger(R.integer.metadata_scope);
                            mScanSuccess = processFile(path, mimeType, this, mScope);
                        }

                        if (isimage) {
                            //解析图片文件
                            mScanSuccess = processImageFile(path);
                        }

                        String lowpath = path.toLowerCase(Locale.ROOT);
                        boolean ringtones = mScanSuccess && (lowpath.indexOf(RINGTONES_DIR) > 0);
                        boolean notifications = mScanSuccess &&
                                (lowpath.indexOf(NOTIFICATIONS_DIR) > 0);
                        boolean alarms = mScanSuccess && (lowpath.indexOf(ALARMS_DIR) > 0);
                        boolean podcasts = mScanSuccess && (lowpath.indexOf(PODCAST_DIR) > 0);
                        boolean music = mScanSuccess && ((lowpath.indexOf(MUSIC_DIR) > 0) ||
                            (!ringtones && !notifications && !alarms && !podcasts));
						//将扫描结果更新进入数据库  这个是支持批量插入的。
                        result = endFile(entry, ringtones, notifications, alarms, music, podcasts);
                    }
                }
                if (mNeedProgressUpdate) {
                    // Update scan progress after each file scan.
                    scanProgressUpdate();
                }
            } catch (RemoteException e) {
                Log.e(TAG, "RemoteException in MediaScanner.scanFile()", e);
            }
//            long t2 = System.currentTimeMillis();
//            Log.v(TAG, "scanFile: " + path + " took " + (t2-t1));
            return result;
        }
```

### 音视频文件的解析

音视频的解析调用的是一个JNI方法processFile，而它的内部经过层层调用会来到TsStagefrightMediaScanner#processFile  而C++层的processFile会调用processFileInternal来解析音视频文件。

在processFileInternal会通过StagefrightMetadataRetriever来解析相关的音视频信息。但是需要注意的是这里并没有音频专辑图片和视频抽帧图。

### 图片文件的解析

图片的解析相对而言比较简单，就是在不获取实际图片的情况下获取图片的宽高信息

```java
//采样率是1  即为原始图片
mBitmapOptions.inSampleSize = 1;
//不获取图片的宽高
mBitmapOptions.inJustDecodeBounds = true;

private boolean processImageFile(String path) {
    try {
        mBitmapOptions.outWidth = 0;
        mBitmapOptions.outHeight = 0;
        BitmapFactory.decodeFile(path, mBitmapOptions);
        mWidth = mBitmapOptions.outWidth;
        mHeight = mBitmapOptions.outHeight;
        return mWidth > 0 && mHeight > 0;
    } catch (Throwable th) {
        // ignore;
    }
    return false;
}
```

# 音频专辑图片的处理

音频专辑图片通过openFile的方式打开



# 思考如何提升扫描效率？

1. 以u盘为单位，对数据进行存储，每次开始扫描对比值需要对比当前U盘的数据（从目前的代码流程来看，它应该是所有的数据存在同一个数据库的表，在多看看代码）。**如果拔出U盘清除数据库，应该就没有这个事了**
2. 多线程扫描，每一个u盘的扫描还是通过HandlerThread来进行分发，但是在文件夹的扫描过程中是否可以多个文件夹并行处理？代码难度比较高。会涉及到C++层和java层的代码配合更改。



# 如何实现快速播放功能

1. HMI保存lastMode信息，hmi直接尝试播放 lastmode 
2. MediaProvider保存LastMode信息  在扫描开始的时候直接先匹配判断LastMode 是否被更改，不存在则删除lastMode，有没有可能存在hmi查询了lastMode 但是lastMode信心还没有对比完成？需不需要在同步完成lastMode信息后发送消息？还是说使用同步机制来保证查询到的lastmode是可用的。

# 问题

在拔出U盘删除扫描入库的数据是否合理？删除的时候LastMode信息怎么来进行维护？解码出来的音频图片是否需要删除？

应用层插入数据库的数据如何维护？是否会随着使用时间一直增加，谁来删除不在使用的数据？MediaProvider根据什么规则来删除

U盘拔出的时候删除数据库表，历史记录如何和原来表中的数据对应？通过path?至少上一个U盘的数据应该做对应的保留吧？



文件夹查询？如何定义Uri

- 查询某个文件夹下的子文件夹
- 查询包含音频/视频/图片 的文件夹
- 查询某个文件夹下的文件