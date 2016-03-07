---
layout:     post
title:      "Handler、Message、MessageQueue随笔"
subtitle:   "内部交流分享"
date:       2016-03-07
author:     "Chenfeiyue"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Android
    - 分享
    - Blog
---


## Handler、Message   
### 1、基本用法：   
创建Handler重写handlerMessage（Message msg）处理消息 

```
Handler handler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        // TODO 处理消息
    }
};

Handler handler = new Handler(new Handler.Callback(){
    @Override
    public boolean handleMessage(Message msg) {
        // TODO 处理消息
        return false;
    }
});
```

```
handler.sendMessage(msg);
handler.post(new Runnable(){...});

```

### 2、主线程默认已经创建Looper，无须重复创建
ActivityThread.java

```
public static void main(String[] args) {
    Looper.prepareMainLooper();
...
    Looper.loop();
...
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

## Handler解析
Handler负责把消息放入线程的消息队列中以及分发消息。
### 1、创建Handler
创建Handler之前需要创建Looper，否则抛出throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");


```
/*
 * @param callback The callback interface in which to handle messages, or null.
 * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
 * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.  
 *
 * @hide
*/
public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();// 使用当前线程所在的Looper
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has 
            not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;  //标志Message是否为异步Message.setAsynchronous
}
```

```
/*
* @hide   
*/
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;    // 为handler指定Looper
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

```

### 2、发送消息到消息队列MesageQueue

Handler中提供了很多个发送消息的方法，其中除了sendMessageAtFrontOfQueue()方法之外，其它的发送消息方法最终都会辗转调用到sendMessageAtTime()方法中

![发送消息](http://img.blog.csdn.net/20160303001436631)
 
（1）boolean sendEmptyMessage(int);
// 发送一条消息到队列，成功返回true，失败返回false，通常是因为Looper已经退出

（2）sendEmptyMessageAtTime(int, long)指定时间发送消息 相对时间
SystemClock.uptimeMillis() + 3000; // 从开机到现在的毫秒数（手机睡眠的时间不包括在内）；  

（3）通过sendMessageAtFrontOfQueue()方法来发送消息的，它也会调用enqueueMessage()来让消息入队，只不过when为0，这时会把mMessages(MQ头部消息)赋值为新入队的这条消息，然后将这条消息的next指定为刚才的mMessages，这样也就完成了添加消息到队列头部的操作。 
  
（3）
![post（Runnable）](http://img.blog.csdn.net/20160303001511603)

Post Runnable 到消息队列。将Runnable转换为Message

```
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r; // msg的callback指向Runnale
}
```

### 3、消息队列中移除消息
![移除消息](http://img.blog.csdn.net/20160303001534260)
 
一般在Activity页面结束时调用handler.removeCallbacksAndMessages(null);
移除队列中所用的callbacks和messages。详见MessageQueue. removeCallbacksAndMessages();

### 4、消息事件处理
**Handler 里面的mLooper所在的线程决定了 handleMessage 方法所在的线程**

message的处理比较简单，先判断Message中有没有指定的callback对象（Runnable），有的话就调用callback的run方法，没有则调用我们自己创建Handler对象时实现的handleMessage(Message msg)方法，就这样实现了消息的分发。

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {  
    // 处理Runnable消息调用runnable.run();
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

###5、子线程中创建Handler  
在一个子线程中创建Handler时，必须初始化该线程的Looper对象，因为普通的Thread默认是没有消息队列的。

```	
class MyThread extends Thread {
    public Handler mHandler;
	public void run() {
		Looper.prepare();
		mHandler = new Handler() {
			public void handleMessage(Message msg) {
				/* 处理接收到的消息 */
			}
		};
		Looper.loop();
	}
}
```

### 6、Handler的特点
1.handler可以在任意线程发送消息，这些消息会被添加到关联的MQ上。
2.handler是在它关联的looper线程中处理消息的。
3.**一个线程可以有多个Handler，但是只能有一个Looper和一个MessageQueue！** 

Q？
同一个线程中的所有消息是否共享MQ？
MQ如何分发到不同的Handler处理消息？
A：是
handler.target

## Message 解析
Message本身是一个Parcelable对象
### Message 可以传递的参数有：
1. arg1 arg2 整数类型，是setData的低成本替代品。传递简单类型
2. Object 类型 obj(Parcelable)
3. what  用户自定义的消息代码，这样接受者可以了解这个消息的信息。每个handler各自包含自己的消息代码，所以不用担心自定义的消息跟其他handler有冲突。
4. 其他的可以通过Bundle进行传递
Message可以通过new Message构造来创建一个新的Message,但是这种方式很不好，不建议使用。最好使用Message.obtain()来获取Message实例,它创建了消息池来处理的。
### 1、	创建Message

```
Message msg = new Message();     (不要这样写)

Message msg = handler.obtainMessage();
Message msg = Message.obtain();
// 这两种方式的区别
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h; // 指定Message的句柄（消息处理者）Handler
    return m;
}
```

```
/**
* Return a new Message instance from the global pool. Allows us to
* avoid allocating new objects in many cases.
*/
// 从缓存池中构建一个Message，如果sPool为空说明没有缓存的Message，则新建一个
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0;   // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

### 2、Message的回收缓存

```
/**
* Recycles a Message that may be in-use.
* Used internally by the MessageQueue and Looper when disposing of queued Messages.
*/
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE; // 标示为in_use_flag
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        // 最多可以缓存50个Message
        if (sPoolSize < MAX_POOL_SIZE**（50）**) {
            // 将缓存的sPool指向当前msg，next指向原有的sPool
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

在一个无限for循环中遍历消息队列，然后调用Handler进行消息分发处理，分发之后调用recycleUnchecked ()把Message对象回收到Message Pool中（最大值为50个，若消息池中已经有50个Message，则不做缓存）

例：
MQ----> m1,m2,m3; // 未处理的message
m1.recycleUnchecked();
![这里写图片描述](http://img.blog.csdn.net/20160303000358680)

m2.recycleUnchecked();  
![这里写图片描述](http://img.blog.csdn.net/20160303001204549)

m3.recycleUnchecked();
![这里写图片描述](http://img.blog.csdn.net/20160303001626307)

Message.obtain();
![这里写图片描述](http://img.blog.csdn.net/20160303001704620)

### 3、msg. markInUse(); 
// 标记msg为FLAG_IN_USE，加入MQ时做检查（此状态的msg不能多次加入MQ），调用Message.obtain()清除flags；
### 4、将一个消息设置为异步
Message.setAsynchronous(Boolean async) // 详见“MessageQueue同步分割栏”

## Looper解析
### 1、Looper.prepare()

```
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}
```

创建一个Looper对象，它的内部维护了一个消息队列MQ。注意，一个Thread只能有一个Looper对象，不能多次调用Looper.prepare()，否则将抛出异常。

### 2、Looper.loop()
```
public static void loop() {
    final Looper me = myLooper();  // 获取当前线程所在                                                   Looper，不能为空
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) { // 循环从MQ中取出消息,没有消息则阻塞
        Message msg = queue.next(); // might block   
        if (msg == null) {
        // No message indicates that the message queue is quitting.
            return;
        }
        // 交给相关Handler处理消息
        **msg.target.dispatchMessage(msg);**   
        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        msg.recycleUnchecked();   // 消息回收
    }
}
```

### 3、Looper.quit()  quitSafely();

调用mQueue.quit();

当我们调用Looper的quit方法时，实际上执行了MessageQueue中的removeAllMessagesLocked方法，该方法的作用是把MessageQueue消息池中所有的消息全部清空，无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送的需要延迟执行的消息）还是非延迟消息。

当我们调用Looper的quitSafely方法时，实际上执行了MessageQueue中的removeAllFutureMessagesLocked方法，通过名字就可以看出，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理，quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。

无论是调用了quit方法还是quitSafely方法只会，Looper就不再接收新的消息。即在调用了Looper的quit或quitSafely方法之后，消息循环就终结了，这时候再通过Handler调用sendMessage或post等方法发送消息时均返回false，表示消息没有成功放入消息队列MessageQueue中，因为消息队列已经退出了。
需要注意的是Looper的quit方法从API Level 1就存在了，但是Looper的quitSafely方法从API Level 18才添加进来。
### 4、其他方法
getMainLooper() 获取主线程Looper
myLooper() 获取当前线程Looper
getQueue() 获取MQ
getThread() 获取当前Looper所在线程

## MessageQueue
MessageQueue是一个按照when大小排列的链表结构。
### 1、enqueueMessage(Message msg, long when)
同一个没有被处理的message不能多次加入队列

```
// 添加消息到消息队列, 最终的mMessages是按照when的由小到大排列
boolean enqueueMessage(Message msg, long when) {
    // 检查msg合法性，必须包含handler且非FLAG_IN_USE状态
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //如果已经调用MessageQueue.quit，                                      那么不再接收新的Message    
            msg.recycle();
            return false;
        }
        msg.markInUse();// msg  in_use_flag
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        // 队列为空或msg.when == 0或 msg.when < mMessages.when时
        // 将msg直接插入到队列头部
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
        // Inserted within the middle of the queue.  Usually we don't have to wake
        // up the event queue unless there is a barrier at the head of the queue
        // and the message is the earliest asynchronous message in the queue.
        
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            // 根据when的大小顺序，插入到合适的位置
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                // 如果在插入位置以前，发现异步消息，则不需要唤醒
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);  //唤醒nativeMessageQueue
        }
    }
    return true;
}
```

nativeWake，和natePollonce的作用：
　　nativePollOnce(mPtr, nextPollTimeoutMillis);暂时无视mPtr参数，阻塞等待nextPollTimeoutMillis毫秒的时间返回，与Object.wait(long timeout)相似
　　nativeWake(mPtr);暂时无视mPtr参数，唤醒等待的nativePollOnce函数返回的线程，从这个角度解释nativePollOnce函数应该是最多等待nextPollTimeoutMillis毫秒

### 2、removeMessage(int what);

```
//删除所有what 和obj = object 的msg
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }
    synchronized (this) {
        Message p = mMessages;

        // Remove all messages at front.
        // 循环移除MQ 队列头部所有符合要求的Message
        while (p != null && p.target == h && p.what == what
                  && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }

        // Remove all messages after front.
        // 循环移除MQ中间所有符合要求的Message
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    // 移除message并交换位置
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

### 3、Message next()

```
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration   空闲handler数量
    int nextPollTimeoutMillis = 0;   // MQ阻塞时间
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);   //MessageQueue阻塞nextPollTimeoutMillis 指定时间

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();  // 开机相对时间（不包含休眠时间）
            Message prevMsg = null;
            Message msg = mMessages;

		    // 遇到同步分隔栏，忽略该消息，取下一个 异步消息
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {     // 遇到延迟消息，则阻塞一段时间 nextPollTimeoutMillis
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.  // 得到Message，从MQ里移除此消息
                    mBlocked = false;
                    if (prevMsg != null) {    // prevMsg != null,说明是同步分隔栏消息，
                        prevMsg.next = msg.next;  // 保留MQ头部为同步分隔栏消息（为了取出下一个异步消息），替换next消息
                    } else {
                        mMessages = msg.next;   // 不包含同步分隔栏消息，替换当前head为next消息
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

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                    // 线程空闲，计算IdleHandler的数量
                pendingIdleHandlerCount = mIdleHandlers.size();
            }

            // 没有IdleHandler   阻塞队列
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
        // 处理IdleHandler部分
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            // 是否需要移除IdleHandler
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

### 4、MessageQueue.IdleHandler

```
messageQueue.addIdleHandler(new MessageQueue.IdleHandler() {
    /**
     * 返回值boolean 意思是needKeep
     * true，表示要保留保留， 代表不移除这个idleHandler，可以反复执行
     * false代表执行完毕之后就移除这个idleHandler, 也就是只执行一次
     */
    @Override
    public boolean queueIdle() {
        Log.e(TAG, "-------------->queueIdle  主线程空闲了");
        return true;
    }
});

```

### 5、同步分割栏

也是一个targer = null 的Message

“同步分割栏”是起什么作用的呢？它就像一个卡子，卡在消息链表中的某个位置，当消息循环不断从消息链表中摘取消息并进行处理时，一旦遇到这种“同步分割栏”，那么即使在分割栏之后还有若干已经到时的普通Message，也不会摘取这些消息了。请注意，此时只是不会摘取“普通Message”了，如果队列中还设置有“异步Message”，那么还是会摘取已到时的“异步Message”的。
在Android的消息机制里，“普通Message”和“异步Message”也就是这点儿区别啦，也就是说，**如果消息列表中根本没有设置“同步分割栏”的话，那么“普通Message”和“异步Message”的处理就没什么大的不同了**。

**int postSyncBarrier(long when)**  // 往MQ里加入一个同步分割栏，按照when的大小插入到合适位置
**removeSyncBarrier(int token)** // 从MQ中移除同步分割栏

### 6、清空MQ

```
void quit(boolean safe) {
    if (!mQuitAllowed) {   //UI线程的Looper消息队列不可退出 mQuitAllowed = false
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) { // 移除MQ所有的延迟消息
            removeAllFutureMessagesLocked();
        } else {    // 移除MQ中的所有Message
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr); // 唤醒MQ
    }
}
```

```
// 移除MQ中的所有Message
private void removeAllMessagesLocked() {
    Message p = mMessages;
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
```

```
// 移除MQ所有的延迟消息 n.when > now
private void removeAllFutureMessagesLocked() {
    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;
    if (p != null) {
        if (p.when > now) {   // 如果队列头部消息为延迟消息，则清空MQ
            removeAllMessagesLocked();
        } else {
            Message n;
            for (;;) {
                n = p.next;
                if (n == null) {
                    return;
                }
                if (n.when > now) {  // 遍历找到延迟消息，退出循环
                    break;
                }
                p = n;
            }
            p.next = null; 
            // 清空所有的延迟消息
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }  
}
```

## Handler-memory-leak
http://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html
## 参考：
http://bbs.9ria.com/thread-247435-1-1.html
http://www.mamicode.com/info-detail-984722.html
http://www.cnblogs.com/codingmyworld/archive/2011/09/14/2174255.html
http://my.oschina.net/youranhongcha/blog/492591?fromerr=d6t15a3t#OSC_h3_10
