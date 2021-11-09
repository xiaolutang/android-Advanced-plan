# 什么是MessageQueue 同步屏障？

看到MessageQueue 同步屏障，很多人或许都会问什么是MessageQueue 同步屏障？其实它就是我们常说的消息机制同步屏障。

# 如何添加屏障消息

MessageQueue提供了一个方法postSyncBarrier 来给MessageQueue添加同步屏障

```java
private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                //查找到比当前消息时间大的message
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

