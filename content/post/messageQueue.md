---
title: "MessageQueue"
date: 2022-06-08T21:19:18+08:00
lastmod: 2022-06-08T21:19:18+08:00
keywords: []
description: ""
tags: [Android]
categories: []
author: "iri.jwj"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

# Android 消息循环机制详解

Android 的消息循环机制中，最关键的类就是 Handler、MessageQueue、Looper、Message 这几个。本文中将从源码的角度详细地分析 Android 消息循环机制的实现方式（将从 send 、 remove 这两个流程分析），也将会涉及到 Native 层的部分。

![](/messageQueue/消息循环机制类图.png)

## 创建

首先，在主线程中我们使用 Handler 时，可以直接使用下面的代码：

```java
Hanlder hanlder = new Handler();
```

调用无参构造函数可以直接创建 handler， 但是在子线程中只使用上述代码将会抛出异常。现在来看看 Handler 的构造函数。

#### Handler()

```java
    public Handler(Callback callback, boolean async) {
        ······
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;// 这里的 async 影响了通过这个 Handler 发送的 Message
    }
```

Handler 的无参构造函数最终调用的时上述构造函数，其中抛出异常的前一句就是关键，通过 Looper.myLooper() 获取的将会是绑定在当前线程的 Looper，当没有调用 Looper.prepare() 时获取的 Looper 为空。

#### Looper.myLooper()

```java
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

这里的 sThreadLocal 是一个静态变量，get() 方法实现了获取当前线程对应的 Looper。如下

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

有 get，就应该也有 set。set 方法的调用在 Looper.prepare() 里完成。

```java
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }//不允许重复创建
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

当 sThreadLocal 中不存在对应的 Looper 时，就创建新的 Looper 并保存到 sThreadLocal 中。同时不允许重复创建，当一个线程对应的 Looper 已存在时再调用会报异常。

看到这里，可以发现在新线程中只要调用了 Looper.prepare() 方法，然后再创建 Handler 将是没有问题的。但是我们在主线程中创建 Handler 时，并不需要去调用 Looper.prepare() 方法，这是因为在 ActivityThread 主线程中完成了对主线程 MainLooper 的初始化。如下：

#### ActivityThread.main()

```java
 public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        ·······
        Looper.prepareMainLooper();
        ······
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
       ······

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

这里调用的方法是 Looper.prepareMainLooper() 方法，跟上面调用的不同的就是，这里创建的 Looper quitAllowed 为 false，毕竟这里创建的是主线程的 Looper，自然就不允许主动调用 quit 方法了。此外， prepareMainLooper 中也会将当前主线程的 Looper 保存到 sMainLooper 静态成员中去。这就实现了在子线程中可以通过 Looper.getMainLooper() 来获取主线程中的消息队列，向主线程中发送消息。

#### Looper(boolean)

```java
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

Looper 的构造函数中创建了 MessageQueue，同时将当前的线程信息保存起来。

#### MessageQueue(boolean)

```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

MessageQueue的构造函数最关键的是调用了 nativeInit() 方法，在 Native 层同样创建了一个 NativeMessageQueue，返回 NMQ 的内存地址。

## 消息入队

Handler 发送消息有许多方法，诸如 post*()，sendMessage\*()等方法，但这些方法最终都调用到的是 sendMessageAtTime(Message,long) 方法：

#### Handler.sendMessageAtTime(Message,long)

```java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

#### Handler.enqueueMessage(MessageQueue,Message,long)

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这里设置了 Message 的 target 属性为自己，在处理消息时会使用到。最后调用了 MessageQueue.enqueueMessage(Message,long)方法。

#### MessageQueue.enqueueMessage(Message,long)

```java
boolean enqueueMessage(Message msg, long when) {
        ·······
        synchronized (this) {
           ·······
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                //这个地方直接插入了链表的头部
                msg.next = p;
                mMessages = msg;
                //当 Message 链表为空且 idleHandler 为空时，mBlocked = true
                needWake = mBlocked;
            } else {
                //只有队列被 Barrier 阻挡了，且新 message 也是个异步消息
                //且新 message 要被加入到队头时，needWake = mBlocked
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    //如果不是加在队头，将 needWake 重新置为 false
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

这里根据 Message.when，选择新 Message 的插入位置。当消息队列原本为空时，应该需要唤醒线程，调用了 nativeWake(mPtr) 方法。

## 消息出队

### 消息的正常执行

出队的逻辑在 Looper.loop() 方法里

#### Looper.loop()

```java
    public static void loop() {
        final Looper me = myLooper();
        ······
        final MessageQueue queue = me.mQueue;
        ······
        boolean slowDeliveryDetected = false;

        //死循环，一直从消息队列中读取下一个消息
        for (;;) {
            // 从消息队列中获取消息，消息的阻塞在这里实现
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            ······//省略掉了大部分的时间检测，判断 Message 的分发处理是否超时
            try {
                // msg.target 获取的就是将这个 msg 发送过来的 hanlder
                msg.target.dispatchMessage(msg);
                //通知观察者
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                //记录分发的结束时间
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
            ······
            msg.recycleUnchecked();
        }
    }
```

下面按顺序来看看 MessageQueue.next() 方法 和 Handler.dispatchMessage() 方法

#### MessageQueue.next()

```java
    @UnsupportedAppUsage
    Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {
            return null; //当 messageQueue quit 之后，mPtr 会置为0.
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                //当MessageQueue中下一个消息的开始时间较晚时
                //先处理Binder中的消息，以免处理Message时间过长导致 Binder 阻塞
                Binder.flushPendingCommands();
            }
            //native 方法，这里暂时不做了解
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    //当第一个 Message 是 barrier 时，表明处于一种被阻挡的状态
                    // 只会返回被标记为 Asyn 的 Message
                    // 因此现在遍历找到第一个异步 message
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    // 当 message 不为空时，判断 message 的处理时间是否已经到了，
                    if (now < msg.when) {
                        //计算下一次唤醒时机
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

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                //idleHandler 是用来在消息队列为空
                //或下一个消息触发的时间未到时
                //调用IdleHandler.enqueueIdle()来对空闲时的事务处理，例如 GC.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    //如果这时候连 idleHandler 都没有了，那就进入休眠吧
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; 

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
            //当处理完所有的 idleHandler 后
            //重置下面两个参数以保证在处理 idle 时
            //加入/到期的 message 能够被及时地处理
            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
        }
    }
```

nextPollTimeoutMillis 可能的值为 -1（当队列中没有 message 时），0（当有消息进入空队列，或者处理完 idleHandler后），>=0(处理消息时，计算when-currentTime)

#### Handler.dispatchMessage()

```java
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            //首先检查了 message 中是否有 Runnable 成员
            //handleCallback 中直接调用了 Runnable.run()
            handleCallback(msg);
        } else {
            //当 message 中的 Runnable 为空时，再检查
            //Handler对象中是否注册了 Callback,Callback 是Handler类中定义的接口，声明了
            //handleMessage(Message)方法
            if (mCallback != null) {
                //根据 callback 的返回值判断是否还需要调用Handler.handleMessage()方法
                //handleMessage默认为空实现，需要手动继承后重写方法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

到此消息循环机制中的消息分发部分就结束了，主要包括三个部分。

- Looper 中循环调用 MessageQueue.next() 方法获取 Message，并调用 Message.target.dispatchMessage(Message) 方法。Looper中还完成了对消息处理时，时间处理超时检测，打出Log消息
- MessageQueue.next() 中阻塞的实现在 native 层完成。这里主要完成了消息队列的维护，当消息队列为空时会处理 IdleHanlder 事件。
- Handler.dispatchMessage(Message) 是最简单的方法，Message 的 Runnable 成员是最高优先级的，其次是Handler中注册的 Callback，最后才到 Handler 自身的 handleMessage(message)  方法

### 消息的 remove

消息的弹出包含了另一种方式，就是消息的 remove，通过 remove* 方法将仍在队列中的 message 移除。

#### Handler.remove**()

```c++
    public final void removeMessages(int what) {
        mQueue.removeMessages(this, what, null);
    }
    public final void removeMessages(int what, @Nullable Object object) {
        mQueue.removeMessages(this, what, object);
    }
    public final void removeCallbacksAndMessages(@Nullable Object token) {
        mQueue.removeCallbacksAndMessages(this, token);//除了这个
    }
    public final void removeCallbacks(@NonNull Runnable r) {
        mQueue.removeMessages(this, r, null);
    }
    public final void removeCallbacks(@NonNull Runnable r, @Nullable Object token) {
        mQueue.removeMessages(this, r, token);
    }
```

可以看到 Handler 中每一个 remove 方法最终调用的都会是 MessageQueue.removeMessages() 方法。下面我们来看看这些方法的具体实现

#### MessageQueue.remove**()

```c++
    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }
        synchronized (this) {
            Message p = mMessages;
            // 从链表头部开始，移除位于头部的 message
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }
            // 移除不在头部的 message
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

所有的 remove 方法实现逻辑基本相同。这里是根据 what 和 obj 属性去判断 Message 是否需要移除，对应另外的方法有根据 Runnable  去判断。此外还有一个方法 removeCallbacksAndMessages() 则是仅根据 obj 属性来判断是否需要移除，因此当 obj == null 时，就会移除所有的messages。

## 消息队列的退出

消息队列的退出大概有两种方式，第一是调用 Looper.quit() 方法，第二是等待 GC 时的 finalize() 方法调用。下面就分析一下第一种方式的过程，将会以主线程为例：

在 ActivityThread.H 内部类中，handleMessage方法里调用了 Looper.quit()

```java
case EXIT_APPLICATION:
    if (mInitialApplication != null) {
        mInitialApplication.onTerminate();
       }
    Looper.myLooper().quit();
    break;
```

从case 的条件中可以看出时在退出应用时调用的。

Looper 的quit()方法最终调用到的是 MessageQueue.quit(true)

#### MessageQueue.quit(boolean)

```java
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```

不过这里有一点奇怪的，在创建主线程消息队列时，mQuitAllowed = false，那么直接调用到这里就会抛出异常

两个remove***() 方法主要是将还未执行的 Message 踢出队列。nativeWake() 主要完成的是唤醒线程，从而让 next()  方法可以继续执行，当 next() 方法检测到 mQuitting == true 时，调用 dispose() 方法并返回了 null。到此就完成了 Looper 的退出。

dispose() 方法中也调用了 nativeDestory() 方法:

#### NativeMessageQueue::nativeDestory

```c++
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);//1
}
```

在 decStrong() 方法中，实现了引用计数减1，其中当引用计数为1时，调用 delete() 方法释放空间。

## IdleHandler 与 Barrier

IdleHandler 的设计是为了当 MessageQueue 空闲时，可以不直接进入阻塞，在这之前先完成部分 IdleHanlder 任务，包括 GC任务、广播发送、ActivityThread 中调用native方法清理 pendingResource() 等。

```java
    //ActivityThread中的
	//触发gc
	final class GcIdler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
            purgePendingResources();
            return false;
        }
    }

	//资源清理？
    final class PurgeIdler implements MessageQueue.IdleHandler {
        @Override
        public boolean queueIdle() {
            purgePendingResources();
            return false;
        }
    }
```

#### ViewRootImpl 与 Barrier

```java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 发送 barrier
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            ·······
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            // 移除 barrier
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

           ······
        }
    }
```

barrier 的设计是为了让消息队列在即将进行 view 更新时阻止消息队列处理消息以免导致 view 绘制卡顿。下面看看postSyncBarrier() 方法

#### MessageQueue.postSyncBarrier()

```java
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        synchronized (this) {
            final int token = mNextBarrierToken++;// 每个 barrier 都有一个对应的 token，
            // 当需要移除时就通过 token 找到对应的 message 进行移除
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;//token 保存在了 arg1 里

            Message prev = null;
            Message p = mMessages;
            //下面就是把 barrierMessage 插入正确的位置
            if (when != 0) {
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

这里主要完成了插入 barrier，判断 barrier.when，找到合适的插入点。所有在 barrier 之后的没有被标记为 asyn 的Message 都会被阻止执行，但 asyn 的 Message 不受影响。

总的来说，IdleHandler 机制主要保证的是当消息队列为空或者下一个消息没到时间的话，消息队列暂时不进入阻塞状态，而是去处理一下其它任务，例如 GC 任务和资源清理任务。

Barrier 机制保证的是当进行 view 绘制时，Message 不会影响到 view 绘制的时机。

## Native 层解析

native 方法总览：

- private native static long nativeInit();  // 初始化
- private native static void nativeDestroy(long ptr);   //线程销毁
- private native void nativePollOnce(long ptr, int timeoutMillis);   // 处理 native 层的消息，如果没有消息处理则会进入阻塞休眠
- private native static void nativeWake(long ptr); //唤醒
- private native static boolean nativeIsPolling(long ptr); //判断线程是否仍在运行
- private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events); // 向 native 层注册文件描述符，epoll 会监听

### 初始化：


前文中我们知道，在 MessageQueue 创建时，会调用 nativeInit() 方法：

#### android_os_MessageQueue::nativeInit

```c++
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    ······
    nativeMessageQueue->incStrong(env);//强应用计数
    return reinterpret_cast<jlong>(nativeMessageQueue);// 返回NativeMessageQueue对象到 java 层
}
```

下面看一看 NativeMessageQueue 的构造函数：

#### NativeMessageQueue::NativeMessageQueue()

```c++
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    // 从线程中获取当前线程的 Looper。
    mLooper = Looper::getForThread();
    // 第一次进入时，线程 Looper 为空，则重新创建一个Looper 并设置到 ThreadLocalStorage 中
    //保证了一个线程中只会有一个 Looper
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

这里创建的 Looper 虽然与 Framework 层的 Looper 名字相同，但是没啥关联。

#### Looper::Looper(boolean)

```c++
Looper::Looper(bool allowNonCallbacks)
    : mAllowNonCallbacks(allowNonCallbacks),
      mSendingMessage(false),
      mPolling(false),
      mEpollRebuildRequired(false),
      mNextRequestSeq(0),
      mResponseIndex(0),
      mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));// 重置 mWakeEventFd
    AutoMutex _l(mLock);
    rebuildEpollLocked(); // 初始化 epoll，并向 epoll 中注册了 wakeEventFd 等监听
}
```

初始化时在 Looper 里创建了 epollEventFd 以及 mWakeEventFd，在 rebuildEpollLocked() 方法里还有将 mRequests 中的文件描述符注册到 epollEvent 中，但是第一次运行 mRequest 为空，跳过。

### 线程唤醒：

当 MessageQueue.enqueueMessage() 方法中向空队列入队 Message 后（或者其它满足 blocked 的情景） ，会调用 nativeWake() 方法

#### NativeMessageQueue::nativeWake()

```c++
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

上面很清晰地可以看出来调用的将会是 Looper::wake() 方法

#### Looper::wake()

```c++
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    uint64_t inc = 1;
    // 这里实现向 mWakeEventFd 中写入一个 int 数字 1，触发 epoll_wait() 返回，实现线程唤醒
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
    ······ //错误处理
}
```

唤醒的方法就是通过文件描述符来触发。

### 消息的处理以及线程的阻塞：

往后，在 MessageQueue.next() 函数中调用了 nativePollOnce(long) 方法，在这个方法里处理了来自 native 层的消息以及文件描述符触发产生的消息，并通过 epoll 机制实现了线程的阻塞唤醒。

#### NativeMessageQueue::nativePollOnce()

```c++
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}

void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);//没做啥操作，最终调用的是 Looper::pollOnce 
    //这里的 pollOnce 调用到的是 Looper.h 的方法，那里最终调用的是带四个参数的方法
    //pollAll(timeoutMillis, nullptr, nullptr, nullptr);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

#### Looper::pollOnce(int,int\*,int\*,void*\*)

```c++
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    //首先这里的 outFd、outEvents、outData 都为 null
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            // 遍历所有 Response 当 response.request.ident 非负时返回 ident；ident 是 response 的id
            // 这里的遍历处理不知道有什么意义。。
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                ······
                return ident;
            }
        }

        if (result != 0) {
            //初次遍历 result = 0 进入 pollInner()
            ······
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```

#### Looper::poollInner()

```c++
int Looper::pollInner(int timeoutMillis) {
······
    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        //计算下一条信息距离当前时间的差值
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            //当上述差值大于零且 java 传递进来的超时时间小于0（即java 消息队列为空）
            //或者差值小于 java 超时时间，就将超时时间设置为这个差值
            timeoutMillis = messageTimeoutMillis;
        }
······

    //重新对成员初始化
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    //这是个很有趣的地方，epoll_wait 用来等待 mEpollFd 的时间，timeout为超时时间
    //这里的 timeout 会有三种状态，>0 时等待 timeout 的时间，=0 时立刻返回
    //-1 时无限等待
    //返回值有3个意思，大于0表示有事件发生，且为事件个数，等于0为没有事件发生，等待超时
    //小于0表示等待时发生错误
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

        ······//检查了各种异常状态，epoll_error(eventCount < 0 时)，以及 time_out(eventCount=0时)

    //都没问题就遍历 event
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd.get()) {
            // 这里判断的是 epoll 中获取的事件是否时 mWakenEventFd 上的事件
            if (epollEvents & EPOLLIN) {
                //判断是否是一个可读事件，因为这个事件一般在 wake 方法里执行，只会是一个可读事件。
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            /*
            这里的 mRequest 是一个 keyedVector，存储了 fd 和对应的 resquest 的结构体
            结构体里封装了 监控文件句柄相关的上下文信息，例如回调函数
            */
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //将请求存入 responses 中，数据结构是 vector
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // Invoke pending message callbacks.
        // 这里处理的是 Native 的 Message
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        //遍历所有 messageEnvelopes，当 messageEnvelope 时间到了，将其发送到 handler 中处理
        if (messageEnvelope.uptime <= now) {
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
				······
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // 下一个 envelopMessage 还没到，记录下时间
            // 但是在 log 里我就基本没见到有 messageEnvelope 处理的时候。
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    // Release lock.
    mLock.unlock();
    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
			······
           
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            // 处理回调函数之后，返回值如果为0 ，标时不需要再次监视该文件句柄
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

这里主要关注 result 中的赋值以及 mNextMessageUptime 的赋值。

result：

- 最开始进入时赋值为 POLL_WAKE

- 当获取的 eventCount < 0 时变为 POLL_ERROR

- 当 eventCount=0 变为 POLL_TIMEOUT

- 当处理了 envelope 时，变为 POLL_CALLBACK

- 最后当处理了 mResponses 中的 callback 时，变为 POLL_CALLBACK

  其中， envelope 不太理解是干啥的，responses 中的 callback 又是啥。推测当 fd 是唤醒消息时，是不会有下面的 envelope 以及 responses 的，不然都变成 callback 了

mNextMessageUptime：

- 在 Done 标签之后，mNextMessageUptime 首先被赋值为 LLONG_MAX

- 当 envelope.uptime 大于当前时间时，mNextMessageUptime 会被设置为 upTime

  最终使用到也是在 pollInner 的方法开始，用于判断 timeoutMillis 参数的赋值

## 结合 Java 层和 Native 层的流程与总结

### 初始化

![初始化](/messageQueue/init.jpg)

这是初始化的流程，在 MessageQueue 的构造函数中调用了 nativeInit()，在 Native 的世界中创建了对应的 NativeMessageQueue，同时 NativeMessageQueue 也创建了自己的 Looper，这里的 Looper 跟 Java 层的Looper就啥关系都没有了。

Native 层的 Looper 中比较重要的两个东西就是 mEpollFd 与 mWakeEventFd。前者负责监听所有注册的 fd 事件更新，在 epoll_wait() 方法中如果发生了监听的事件就返回对应的结果。后者负责传递来自 Java 层的 wake 事件，也就是当 Java 层的消息循环入队了新的消息时，有可能会触发的 nativeWake() 方法。底层的实现是向 mWakeEventFd 调用 write() 写入 1，触发更新。同时Looper 在处理 wake 事件时会调用 Looper.woken() 方法，其中会调用对应的 read() 方法，read() 方法会将 mWakeEventFd 重新置为0。

![looper-loop流程](/messageQueue/looper-loop.jpg)

![nativeLooper-pollOnce](/messageQueue/nativePollOnce.jpg)

这两个流程分别是 Java 层的 MessageQueue.next() 方法以及 native 层的 Looper.pollOnce() 方法。这两个本属于同一个流程，一起画就太大了。

在 MessageQueue.next() 的处理流程中，主要关注的是 nextPollTimeoutMillis 与 mBlocked 的变化。nextPollTimeoutMillis 在队列为空时为 -1，在处理完所有 idleHandler 后重新置为0，当 msg.when 大于 now 的时候会变为差值。三种状态将影响 Looper.pollInner() 方法的执行状态，为0时 epoll_wait() 方法直接返回超时， 这时只会处理 MessageEnvelope 以及上一次继续保留了的 Response，因此在 Native 层停留的时间不多，会比较快速地返回到 Java 的消息处理中。为 -1 时则是进入了线程阻塞，等待 wake 事件的触发或者是其它监听的 fd 触发事件。

mBlocked 的状态影响了 MessageQueue.enqueueMessage() 中是否调用 nativeWake() 方法。

![handler-sendMessage](/messageQueue/handler_sendMessage.jpg)

发送 Message 的流程

这里最关键的 needWake 的判断，影响了是否调用 nativeWake() 方法。nativeWake 方法的最终实现就是调用 write() 方法向 mWakeEventFd 中写入 1，通知到 epoll_wait() 。
needWake：

- 当新 message 需要插入在队头时，needWake = mBlocked
- 当 message 插入队列中时，needWake = mBlocked && p.target == null && msg.isAsynchronous()，也就是只有被 barrier 阻挡了且新消息标记为异步，且 mBlocked 为 true 时，才会触发 nativeWake()。
- 此外，如果 message 不是第一个异步消息，那么 needWake 会被重新置为 false。

mBlocked 被设置为 true 只在一个地方：

```java
if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
    }
```

也就是当消息队列为空或暂时没有消息处理时，如果也没有 pendingIdleHandler，那么 mBlocked 就会被置为 true，此时也意味着这个线程立刻就要进入阻塞状态了。

## Extension

### Message

Message 在整个 handler 机制中是一个消息承载体的角色，包含了诸如 what:int、arg1:int、arg2:int、obj:Object、data:Bundle、callback:Runnable 属性，用于定义需要传递的消息或执行的行为。

其次，Message 内部维护了一个缓存链表，用于复用 Message，避免了可能出现的大量 Message 被创建到被销毁而触发频繁 GC。其中 Message 持有了一个 sPool:Message 属性，用于指向缓存链表的表头，obtain() 方法内当 sPool 为空则直接返回 new Message()。直到被创建的 Message调用 recycle()/recycleUnchecked() 方法时，如果缓存链表没有满时，被回收的 message 会被插入到链表的第一位，同时 recyclerUnchecked() 方法中将 message 的所有成员变量重新初始化，int 型变为0，对象变为 null。

需要注意的是，Message 的构造函数是空的，obtain() 无参方法返回的是一个啥都没有被赋值的 message，其它有参方法即是将参数赋值到对应的字段，这使得 Handler.postDelayed() 这类型的方法创建出的 message 中，what 字段都是 0，此时如果调用 removeMessage(what:Int) 方法，所有 what 为 0 的都会被清除。

### Native 中的 Fd 与 Epoll

在 Native 层的 Looper 中维护了 mWakeEventFd 与 mEpollFd。fd 是 Linux 中的文件描述符，通过文件描述符可以实现消息的通知与获取能力。

mWakeEventFd 是一个 eventfd 对象，eventfd 中维护这一个计数器，在创建时初始化。其中主要有两个函数：

```c++
	//write 是向 eventfd 中写入数据
ssize_t write(int fd, const void* buf, size_t count)
    //read 从 eventfd 中读出数据，同时会将计数器减去对应的数值
ssize_t read(int fd, void* buf, size_t count)
```

而 mEpollFd 则是 Linux 的 Epoll 机制，epoll 机制中通过 epoll_ctl 添加需要监听的事件句柄，例如在 Looper.rebuildEpollLocked() 方法中就默认向 mEpollFd 中注册了 mWakeEventFd，再通过 epoll_wait() 方法等待来自 Java 的唤醒事件。

### Messenger

在 Message 中存在 replayTo:Messenger、sendingUid:int、workSourceUid:int、data:Bundle，这几个属性。Handler 中存在一个内部类 IMessengerImpl 继承自 IMessenger.stub

这里引入了一个前面都没有讲到的东西，Messenger。根据代码文件中的注释，不难理解 Messenger 是一个基于消息的进程间通信工具，通过在一个进程内创建 Messnger 并绑定了 Handler，再将 Messenger 交给另外一个进程。其实现的本质依然是 aidl上的 Binder 跨进程通信。

## 总结

先上一个 Java 层 + Native 层全家福：

![消息循环机制类图+native](/messageQueue/class_native.png)

总结而言，整个 Android 的消息循环机制分为了 Java 和 Native 两层：

Java 层中：

- Handler 完成了处理消息以及发送消息这两个出入口的任务
- Looper 提供了循环消息处理的能力，持有 MessageQueue 的实例，持续地从 MessageQueue 中提取消息
- MessageQueue 则是 Java 层与 Native 层的连接。在 Java 层这一方，MessageQueue 完成了消息队列的维护。而线程的阻塞与唤醒能力则是下移到了 Native 层实现。

Native 层：

- NativeMessageQueue 本质上是一个连接类，提供了 JNI 的接口，同时内部创建了 Looper 对象，通过 Looper 对象实际完成各类任务
- Looper 则是实际的处理者，通过 epoll 机制实现了线程的阻塞与唤醒，同时 Looper 也支持处理来自 Native 层的 MessageEnvelop，在处理顺序上先处理 MessageEnvelopes，然后处理 epoll 机制监听Fd获取到的 Responses ，处理完这两者后才会回到 Java 层处理 Message。
