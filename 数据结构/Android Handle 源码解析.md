### **Android Handle **源码解析

源码epoll 设计思路  设计模型  异步消息 同步消息   HandleThread   IntentService



消息管理机制--系统层

java main()  独立空间 ，进程挂了不会影响其他的应用程序。

Lanuch --》zygote fork--》jvm虚拟机--》Activity Thread--》

main

Looper.perpareMainLooper();

所有的代码都是再Handler中运行的

Looper.loop();

维持着android app整个框架

handle--》sendmessage--》

优先级队列--》是由单链表组成的



#### 1.对Handle的认识

Handle机制是一个消息管理机制，Handle在Android的整个系统架构中处于核心的位置。所有的点击事件、滑动事件、页面跳转、服务启动等等，都是Handle消息机制驱动的，线程切换只是Handle的一个附属功能，所以做Android移动端研发必须对Handle机制有深入的理解。



#### 2.Handle消息机制的运转流程分析

##### 2.1消息的发送流程源码分析

```java
 	
	//发送消息
   public final boolean sendMessage(@NonNull Message msg) {
        //调用延迟发送方法，延迟时间是0 
        return sendMessageDelayed(msg, 0);
    }

	//发送延迟消息
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        //在某个事件点发送消息  SystemClock.uptimeMillis() + delayMillis这个是延迟时间
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    //在某个时间点发送消息
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //将消息插入消息队列
        return enqueueMessage(queue, msg, uptimeMillis);
    }


    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        //绑定target 这里很关键 牵扯到回调以及handle为什么可能发送消息泄露
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //执行消息队列的插入消息方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }
  
 }

```

* 这是Handle里发送消息的流程，可以发现最终调用enqueueMessage，这个方法就是让消息入队列。

* msg.target = this; 这个很关键，这里将Mssage持有了Handle的引用，如果在Activity创建Handle的位置是非静态的，那么Handle就持有了Activity的应用（非静态内部类持有外部类的引用），所以当Message不释放的情况下，Handle不会释放，也就导致Activity不会释放，这就是不规范使用Handle导致内存泄漏的原因。

* 在Looper里边消息被处理的时候，也是通过这个target.回调的dispatchMessage方法。



最后一行执行的是MessageQueue的enqueueMessage()方法，下面来看这个方法。

```java
boolean enqueueMessage(Message msg, long when) {
    //在这里判断了消息的target不能为空，所以应用层不能使用消息屏障
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
	
    //消息入对的时候，使用同步内置锁synchronized，这样保证了消息入对的线程安全。
    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
		//如果正在退出
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            //释放消息 调用的是：recycleUnchecked方法，只是将消息的属性重置，这个消息会加
            //入到消息缓存里边
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;  //消息队列的 是通过单链表实现，拿到单链表的头部节点
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //发送来的消息，比链表头的消息都靠前
            // New head, wake up the event queue if blocked.
            msg.next = p;  //将msg的next节点 指向原来的链表头节点
            mMessages = msg;	//将msg更新为新的链表头节点
            needWake = mBlocked; 
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
		//插入队列中间。通常我们不必唤醒事件队列，除非队列头部有屏障，并且消息是队列中最早的异步消息。		
            //前面的target 不能为空的判断原因就在这里，target为空是needWake=true的一个条件
            //不允许外部发送的消息，使用消息屏障机制
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                //对链表进行遍历，找到msg的位置
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            //将msg插入链表  根据时间进行排序 的优先级队列
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        //如果唤醒，执行唤醒方法  消息队列休眠了，来消息了就需要
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

通过分析MessageQueue的入队方法可以得到：

* 消息队列是具有优先级的，会根据时间进行排序，是通过单链表来实现的。
* 如何在链表的头部插入节点，以及如何根据时间进行链表插入排序，这里的排序算法就是插入排序。
*  needWake = mBlocked && p.target == null && msg.isAsynchronous();当这个条件判断的是需要唤醒，如果p消息是异步消息，也会执行不唤醒，需要阻塞，直到同步消息的到来。

以上就是消息发送的执行的全部过程。

##### 2.2消息从消息队列消费的过程

消息msg加入到消息队列之后，还有一端在一直干活，那就是Looper的loop方法，这个loop方法是什么时候调用的呢？

```java
public static void main(String[] args) {
    ...

    Looper.prepareMainLooper();

    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        //给Looper设置了一个LogPrinter，我们也可以设置设置一个自己的LogPrinter来获取
        //系统进行消息分发的日志，分析时间就可以知道是否发生过掉帧的问题了
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    //开启消息循环
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

**这是ActivityThread的main方法，是Android应用程序启动调用的**。可以看到启动之后就启动了loop方法，loop方法是一个死循环方法，只要应用程序不退出它就会一直执行。如果loop方法异常停止，系统会抛出一个RuntimeException：Main thread loop unexpectedly exited。

所以loop方法就非常关键了，由于loop方法中通过next方法来获取消息的，先来看MessageQueue的next方法

```java
@UnsupportedAppUsage
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    //如果消息循环已经退出并被处理，则返回此处。如果应用程序在不支持的退出后尝试重新启动循
    //环器，则可能会发生这种情况。
    
    //mPtr 是MessageQueue.cpp的对象指针首地址，也就是MessageQueue.cpp的执行函数句柄
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
		//处理消息等待 调用的 epoll_wait方法来实现等待的
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //msg.target==null 表示屏障消息
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                //被一道屏障挡住了。查找队列中的下一条异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {//还未到执行时间
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    //机选需要等待的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages
            //have been handled.
            //处理完所有待处理的消息后处理退出消息。
            if (mQuitting) {
                dispose();  //会释放 mPtr  
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

next方法分析：

* 上来判断了mPtr是否为空，如果为空直接返回null， mPtr是MessageQueue.cpp的对象指针首地址，也就是MessageQueue.cpp的执行函数句柄。mPtr这参数当执行quit方法会赋值为0
* 然后判断消息是否是屏障消息，如果是屏障消息，那么执行异步消息，知道将全部的异步消息执行完。
* 如果消息还未到执行的时间，通过pollOnce方法（epoll_wait）来执行等待





然后来看Looper的loop方法：

```java
public static void loop() {
    //得到Looper对象
    final Looper me = myLooper();
    //如果looper为空，就抛出去一个异常，这个是不是很熟悉，当在子线程创建Handle的时候，如果不调
    //用Looper.perpare()方法就会看到这个异常，原因就在这里，Looper.perpare()创建了Looper
    //对象
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    if (me.mInLoop) {
        Slog.w(TAG, "Loop again would have the queued messages be executed"
                + " before this one completed.");
    }

    me.mInLoop = true;
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    //确保该线程的身份是本地进程的身份，并跟踪该身份令牌实际上是什么。
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    // Allow overriding a threshold with a system prop. e.g.
    // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
    //允许使用系统道具覆盖阈值。例如adb shell 'setprop log.looper.1000.main.slow 1 
    //&& stop && start'
    final int thresholdOverride =
            SystemProperties.getInt("log.looper."
                    + Process.myUid() + "."
                    + Thread.currentThread().getName()
                    + ".slow", 0);

    boolean slowDeliveryDetected = false;
	//开启死循环
    for (;;) {
        //这里是分析Handle流程的核心点：从消息队列获取消息，前面是往消息队列放置消息，这里是从
        //消息获取获取消息 如果消息队列为空会阻塞：让出CPU执行权让程序休眠，这样可以较少cpu
        //消耗，主要不会发生卡顿以及ANR等，
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            //如果拿到一个空的消息就退出循环了，应用程序也就退出了
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        //拿到Printer 输出日志，上层应用程序可以根据这个日志分析是否发生了耗时操作
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        // Make sure the observer won't change while processing a transaction.
        final Observer observer = sObserver;

        final long traceTag = me.mTraceTag;
        long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
        long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
        if (thresholdOverride > 0) {
            slowDispatchThresholdMs = thresholdOverride;
            slowDeliveryThresholdMs = thresholdOverride;
        }
        final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
        final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

        final boolean needStartTime = logSlowDelivery || logSlowDispatch;
        final boolean needEndTime = logSlowDispatch;

        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }

        final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
        final long dispatchEnd;
        Object token = null;
        if (observer != null) {
            token = observer.messageDispatchStarting();
        }
        long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
        try {
            //还记得在Handle的enqueueMessage()方法绑定了target=this吗，
            //关注target=this这个点，对于我们理解整个执行流程很关键
            //这里不就是拿到消息的target在执行dispatchMessage方法进行消息分发的吗
            msg.target.dispatchMessage(msg);
            if (observer != null) {
                observer.messageDispatched(token, msg);
            }
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } catch (Exception exception) {
            if (observer != null) {
                observer.dispatchingThrewException(token, msg, exception);
            }
            throw exception;
        } finally {
            ThreadLocalWorkSource.restore(origWorkSource);
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (logSlowDelivery) {
            if (slowDeliveryDetected) {
                if ((dispatchStart - msg.when) <= 10) {
                    Slog.w(TAG, "Drained");
                    slowDeliveryDetected = false;
                }
            } else {
                if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                        msg)) {
                    // Once we write a slow delivery log, suppress until the queue drains.
                    slowDeliveryDetected = true;
                }
            }
        }
        if (logSlowDispatch) {
            showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
        }
		
        //这里再次输出log 两次时间的差值就是这次消息执行的耗时
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```

从Looper的loop方法中，可以得到：

* 为什么在子线程直接初始化Handle会抛出"No Looper; Looper.prepare() wasn't called on this thread." 这个异常。
* loop方法是一个死循环，只要应用程序在运行，它就不会被停下来。
* 消息的分发前后，会通过Printer来打印日志，可以通过计算这个差值来分析是否发生了掉帧或者卡顿的问题。
* 消息是通过messageQueue的next方法来获取消息的，并通过之前给消息设置的target 通过diapatchMessage()来分发消息的。

整个流程就到了Handle的diapatchMessage方法







```java
public void dispatchMessage(@NonNull Message msg) {
    //我们并没有给msg的callbak赋值，所以callback==null
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        //如果设置了mCallback 执行if
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //因为我分析的过程是直接new的Handle 所以会执行到这里，handleMessage就是我们重写
        //的回调方法了
        handleMessage(msg);
    }
}
```

到这里就回调了handleMessage方法，我们就可以在这个方法里边刷新UI等做一些逻辑了，整个消息的消费过程到这里就结束了。



#### 3.Handle中的一些关键点

* **loop方法是死循环，并不会影响性能**，因为消息队列是一个带优先级的阻塞队列，当消息队列为空会阻塞休眠，让出cpu的执行权，达到cpu合理利用的目的。
* Looper中的sThreadLocal 成员被static final修饰，是全局唯一的;mQueue是被final修饰的所以每一个Looper持有唯一一个Messagequeue;mThread是被final修饰的在Looper中是唯一的
* Loope是通过prepare方法创建的，并添加到了全局唯一的sThreadLocal中
* **Handle是通过内存共享的方式来达到线程切换的**，在主线程进行Handle的创建的时候，通过Looper.myLooper()方法，绑定了sMainLooper，消息队列是在Looper初始化的创建的，所以在子线程发送的消息，直接发送到了主线程创建的MessageQueue中了，然后消息分发到了主线程，所以消息在那个线程分发，关键要看Looper是在那个线程创建的。
* Message的使用缓存池，所以创建的使用直接使用obtain方法就好，Message采用享元的设计模式
* **MessageQueue：是一个带优先级的阻塞队列，无上限，通过单链表的方式实现的。**
* **消息有三类：同步消息、异步消息、屏障消息。**



以上就是对Handle的分析。

