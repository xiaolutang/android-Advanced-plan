ANR全称 Applicatipon No Response指的是应用程序失去响应，对于ANR的处理很多人的第一反应是通过data/anr/trace.txt下的anr文件配合系统log来进行anr分析。但是有这些消息我们巨额能准确定位anr了吗？

# 什么情况会发生ANR

![](anr发生场景.webp)

需要注意的是Activity的onCreate方法进行耗时操作并不会触发ANR(查看源码并没有找到对应的监测).

# 系统ANR的监听

## 广播超时

无序广播：

```java
final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        BroadcastRecord r;

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "processNextBroadcast ["
                + mQueueName + "]: "
                + mParallelBroadcasts.size() + " parallel broadcasts, "
                + mOrderedBroadcasts.size() + " ordered broadcasts");

        mService.updateCpuStats();

        if (fromMsg) {
            mBroadcastsScheduled = false;
        }

        // First, deliver any non-serialized broadcasts right away.
    //遍历当前的无序广播
        while (mParallelBroadcasts.size() > 0) {
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();

            if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.asyncTraceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_PENDING),
                    System.identityHashCode(r));
                Trace.asyncTraceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_DELIVERED),
                    System.identityHashCode(r));
            }

            final int N = r.receivers.size();
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing parallel broadcast ["
                    + mQueueName + "] " + r);
            //对每个广播接收者进行分发
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Delivering non-ordered on [" + mQueueName + "] to registered "
                        + target + ": " + r);
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            addBroadcastToHistoryLocked(r);
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
                    + mQueueName + "] " + r);
        }
        //省略代码
}
```

有序广播

```java
// Get the next receiver...
int recIdx = r.nextReceiver++;

// Keep track of when this receiver started, and make sure there
// is a timeout message pending to kill it if need be.
r.receiverTime = SystemClock.uptimeMillis();
if (recIdx == 0) {
    r.dispatchTime = r.receiverTime;
    r.dispatchClockTime = System.currentTimeMillis();
    if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
        Trace.asyncTraceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER,
            createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_PENDING),
            System.identityHashCode(r));
        Trace.asyncTraceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
            createBroadcastTraceTitle(r, BroadcastRecord.DELIVERY_DELIVERED),
            System.identityHashCode(r));
    }
    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing ordered broadcast ["
            + mQueueName + "] " + r);
}
if (! mPendingBroadcastTimeoutMessage) {
    long timeoutTime = r.receiverTime + mTimeoutPeriod;
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST,
            "Submitting BROADCAST_TIMEOUT_MSG ["
            + mQueueName + "] for " + r + " at " + timeoutTime);
    //发起Anr监听
    setBroadcastTimeoutLocked(timeoutTime);
}
//... 省略代码
    deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
//... 省略代码
}
```

## Service超时