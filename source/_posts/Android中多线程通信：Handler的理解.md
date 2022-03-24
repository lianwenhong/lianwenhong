---
title: Android中多线程通信：Handler的理解
date: 2022-03-24 23:43:42
categories: Java
---

Android中Handler在我理解主要是为了解决线程间通信。

## Handler机制组成

    使用Android的Handler机制主要要了解几个类：
   
- **Looper**：
    一个线程对应一个或者0个Looper，主线程在ActivityThread的时候会默认创建一个Looper，非主线程中需要先通过Looper.prepare()创建，并且通过Looper.loop()开启。
- **Message**：
    线程间通信的消息载体，Handler利用Message来携带信息给另一个线程，内部包含一个叫sPool的单向链表结构消息缓存池，使用享元模式实现消息的复用防止内存抖动。
- **MessageQueue**：
    消息队列，与Looper一一对应，每个Looper中维护一个消息队列，此消息队列是一个时间优先级队列，内部是一个单向链表
- **Handler**：
    有点像是一个工厂里的机器人，不断从这个线程中发送Message或者是Runnable给另一个线程，并放入MessageQueue中    

所以以上是Handler机制的主要4个要素，每个要素之间的关系很清晰明了：**Handler为生产者，Looper为消费者，MessageQueue为容器，而Message为具体的消息载体。**

<!-- more -->

## Handler使用
如果对Handler还不够熟悉可以看下这2篇文章先熟悉Handler使用：
[Handler实现子线程与子线程、主线程之间通信](https://blog.csdn.net/androidsj/article/details/79816866)
[Handler完全解读——Handler的使用](https://blog.csdn.net/qq_21556263/article/details/82759061)

## Handler运行原理
要深入理解Handler机制我们首先从几个着力点出发
**1.Handler怎么创建的
2.Handler调用sendMessage的时候发生了什么
3.Looper与线程是什么关系
4.Looper.prepare(),Looper.loop()做了些什么事情，为什么如果不执行prepare会报错等。
5.Handler是如何保证多个线程跟同一个线程进行通信时的准确性**

首先从创建一个Handler的的时候构造方法来看，所有构造方法总共有3种参数：
Callback：是一个只有handleMessage的接口，是事件处理的回调，可有可无，后面说
async：是同步消息还是异步消息，这与同步屏障有关，后面说，一般我们都是发的同步消息
Looper：该handler对象所要处理的线程对应的Looper对象，也就是说该Handler的执行都是在looper对应的线程中的

线程分为有消息队列和没有消息队列两种，没有消息队列则线程启动完成执行完操作就结束了 ，而有消息循环队列的时候，线程可以通过循环调度消息队列的方式来执行消息队列中的每一个任务，Android中使用Handler进行线程间通信时，Handler必须指定要通信线程所持有的Looper。
准确的说，其实是Handler需要做的事情是向要通信的线程所持有的消息队列中加入任务。   

```java
public Handler(@Nullable Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    
    mLooper = Looper.myLooper();
    if (mLooper == null) {
    	//这里就是在子线程使用Handler的时候常遇到的异常
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
如果你在Handler构造函数中没有指定Looper，那么调用Looper.myLooper()句柄来获取当前线程所对应的Looper，如果这时该线程还未初始化Looper对象，这时就会抛出一个异常， 这就是为什么在子线程时我们创建Handler前需要先调用Looper.prepare()方法的原因。
跟进Looper.myLooper()看一下：  

```java
public static @Nullable Looper myLooper() {
   return sThreadLocal.get();
}
```
调用Looper.myLooper我们看到其实是从sThreadLocal变量中get一个Looper对象，这样就确保了每个线程都对应了一个Looper，而每个Looper中拥有一个MessageQueue，这样就实现了线程-Looper-MessageQueue这3个部分一一对应的关系。如果对ThreadLocal不熟悉的话可以百度一下，或者等我有空我会更新一篇文章。

既然已经知道了线程-Looper-MessageQueue是一一对应的关系，想与某一线程进行通信只需将消息加入到该线程对应的MessageQueue中就能实现，那么我们看下Looper与MessageQueue做了些什么东西。

Looper.prepare():

```java
public static void prepare() {
	prepare(true);
}

//quitAllowed表示该Looper所对应的消息队列是否可退出，具体的退出在Looper.quit()方法里
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		//表示该线程对应的Looper对象已存在，不能重复调用prepare()，否则就抛异常
	    throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}
```
所以调用prepare()时是创建Looper实例，看下构造：

```java
private Looper(boolean quitAllowed) {
	//创建消息队列
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
所以到这里，线程持有了一个Looper对象，Looper对象中持有了一个MessageQueue队列，一对一。
紧接着消息队列有了，需要让队列开始运作起来，就好比流水线一样，需要轮训这个队列取出消息来执行。而消息的添加是在Handler的sendMessage中我们一会儿再说
Looper.loop():

```java
public static void loop() {
	final Looper me = myLooper();
	if (me == null) {
		//在这里又判断了一下Looper.prepare()是否调用。
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
	final MessageQueue queue = me.mQueue;
	
	// Make sure the identity of this thread is that of the local process,
	// and keep track of what that identity token actually is.
	Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();
	...
	//此处开启一个无限循环，只有当msg == null时才会退出，
	for (;;) {
		//调用MessageQueue内部的next()方法，此方法在消息队列为空的时候会执行线程阻塞，阻塞原理是使用Linux的epoll实现的
		Message msg = queue.next(); // might block
		if (msg == null) {
		    // No message indicates that the message queue is quitting.
		    return;
		}
		...	
		try {
			//调用handler的dispatchMessage方法来处理消息，最后真正执行到handler的handleMessage()
			msg.target.dispatchMessage(msg);
		    ...
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
		...
		//使用享元模式来将该Message对象还原并放回消息池中以便将来复用减少内存抖动，具体实现在Message中
		msg.recycleUnchecked();
	}
}
```

省略部分与主流程关系不大的代码，我们看下注释中的各个方法

首先在loop()时也会做一遍prepare()的调用校验，因为在开启消息队列之前必然是要确保Looper和消息队列已存在，否则一切都是空谈。

紧接这开启了一个无限循环，只有在消息为空的时候才把该循环退出表示该线程的MessageQueue已经不再需要。

在queue.next();方法中，如果取不到消息的话，会执行Linux的epoll方法来阻塞住当前线程，以便有消息被加入到消息队列时能获取到并执行的同时，在没消息的时候又不会让线程一直在做无用的轮训消耗性能。
可能我们会不理解的是，上面明明说没有消息的时候会退出for循环这里又说会阻塞，这是怎么回事？
其实是这样的，首先我们要确保一个线程的MQ一直在等着消息进来从而执行消息，而如果直接在取不到的时候直接结束消息队列的循环显然是不符合的。那么底层的实现就是首先先把线程阻塞住等待消息加入，而什么时候会唤醒线程呢，1个是当消息被加入到MQ的时候，一个是当调用了quit()来表示此轮训机制已不再需要的时候。所以for循环里的msg == null这个判断其实针对的就是第2种情况。

如果成功取到了一个消息，那么就去执行它，就调用dispatchMessage()方法最终会调用到Handler的handleMessage()方法。
这里思考一下，通过handler调用sendMessage()，而从MQ中取出消息的时候，怎么能定位到发送消息的handler中去执行处理呢？其实这里就是通过Message这个载体把发送消息的Handler保存起来，也就是这个target！！！

MessageQueue.next()是如何取消息的

```java
Message next() {
    ...
    for (;;) {
        ...
		//Linux的epoll实现阻塞的入口
        nativePollOnce(ptr, nextPollTimeoutMillis);
		//通过加锁保证多线程出队的正确性，因为一个Handler可以处理多个线程中的消息
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //表示这是个同步屏障，同步屏障与普通消息的区别是同步屏障的target是null
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                    //寻找异步消息来返回
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    //如果这个消息的执行时间还没到，那就不做其他操作，因为这里是无限循环，所以会一直轮训直到时间到
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
                    //取出符合条件的第一个消息返回
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
          }
          ...
    }
}
```
以上在MQ中实现了消息的获取。

看完Looper以及取消息的实现，我们再来看下我们sendMessage()的时候内部做了些什么处理，怎么与Looper中的逻辑关联上的

当调用sendMessage之后，其实最终是调用到以下方法：

```java
//upTimeMillis是参数是延迟执行的时间，例如当postDelayed的时候可以指定一个延迟时间一样
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
	 if (queue == null) {
	     RuntimeException e = new RuntimeException(
	             this + " sendMessageAtTime() called with no mQueue");
	     Log.w("Looper", e.getMessage(), e);
	     return false;
	 }
	 //此时的uptimeMillis=SystemClock.uptimeMillis() + delayMillis
	 //意思是当前系统时间加上开发者传入的所需要延迟的时间
	 return enqueueMessage(queue, msg, uptimeMillis);
}
```
接着调用到MQ中：

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
    long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
	//此处指定该消息为异步消息，异步消息一会儿讲同步屏障的时候再说
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    //进入MQ中的入队操作
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
MessageQueue中：

```java
boolean enqueueMessage(Message msg, long when) {
	//如果没有指定target则不入队抛出异常，其实这里一个是为了后续能定位出需要处理消息的Handler，也为了与同步屏障消息区分开了（这个后说）
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    //如果这个消息已经在入队等待处理往后的流程，那这个消息不能被重复使用直到该消息执行完毕
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
	//因为同一个线程可以与无数个线程进行通信，所以在入队和出队都必须保证同步
    synchronized (this) {
    	//如果正在执行退出操作（也就是Looper.quit()），则不再入队新消息
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
		//
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        //1.如果队列中还没有消息，则添加队列头并唤醒线程去执行轮训那一套流程取出消息并执行
        //2.消息需要立刻执行，则也是唤醒线程执行消息
        //3.消息需要执行的时间点比队头消息时间小，那么表示这个消息需要先执行，也执行唤醒执行消息
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            // 除非队头消息是同步屏障，并且消息是队列中最早的异步消息。否则都不唤醒线程
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                	//这一段的操作在此处退出，目的是将添加进来的消息按时间先后顺序插入到合适的位置。
                    break;
                }
                // 这里屏蔽掉同步屏障消息，也就是说加入一个同步屏障时，不需要唤醒线程
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            // 将消息入队到链表中合适位置
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
        	//唤醒线程
            nativeWake(mPtr);
        }
    }
    return true;
}
```
## Handler小结

看到这里我们的整个流程已经清晰了。
**1.Looper.prepare()创建消息队列并将线程--Looper--MessageQueue进行一一对应**

**2.Looper.loop()开始消息队列循环并通过一套阻塞机制来实现队列的等待以及取消息。最终调用调用MQ.next()，并在无消息时阻塞线程**

**3.通过Handler.sendMessageAtTime()方法发送一个消息，最终调用到MQ.enqueueMessage()方法来将消息入队，并在入队成功唤醒线程**

**4.Looper.quit()方法实现退出Handler机制，通过唤醒线程并且往下执行让msg == null来实现让Looper.loop()中的无限循环退出。**

看完整个流程，开头的5个疑问已全部解决。

## 深入思考：
**1.在主线程中我们没有手动调用过Looper.prepare()方法，却也没见有什么问题?
2.什么是同步消息什么是异步消息？
3.什么是同步屏障，同步屏障的使用场景是什么？
4.我们postDelayed()指定延迟时间时，一定会在指定的延迟时间之后执行吗？**
问题1:
	原因在于那是因为在主线程中Google已经帮我们做了这些操作，不信看看整个Android应用程序的主入口：ActivityThread的main()方法：
	

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

    // Install selective syscall interception
    AndroidOs.install();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");
	//执行Looper初始化
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
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    //开启Looper循环
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
问题2:
	在Handler机制中区分同步消息或者异步消息其实主要是根据Message.setAsynchronous()方法来设置的，设置true为异步消息，false为同步消息，默认我们创建的都是同步消息。
	如何使用Handler发送同步消息或者异步消息，通过Handler的构造函数可以指定async为true来实现，默认是false
```java
public Handler(@Nullable Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    //指定向该handler中发送的消息类型
    mAsynchronous = async;
}
```
问题3:
	同步屏障是一个target = null的Message，Handler机制通过这个方式来区分出同步屏障消息，同步屏障消息并没有真正的执行逻辑，只是为了让异步消息得到更优先的执行。当轮训消息队列的时候碰上同步屏障，则一直往后寻找最早的异步消息来执行。如果没有异步消息就阻塞指导有异步消息被加入队列或者同步屏障被移除为止。同步屏障发送的代码已被hint，如果需要调用则需要反射实现。
	
	MQ中发送同步屏障的方法：
	

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

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
	MQ中移除同步屏障的方法：
	

```java
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```
同步屏障具体开发过程中使用较少，大都是在系统层渲染时使用，如需了解以下是几个链接我觉得说的不错：
[Handler机制——同步屏障](https://blog.csdn.net/start_mao/article/details/98963744)
[handler的同步屏障使用场景](https://blog.csdn.net/cpcpcp123/article/details/115374057)

问题4:
	不一定，因为我们消息入队的时间是系统加上当前时间来入队，只是一个相对顺序，消息的执行是串行的，所以必须等上一个消息执行结束才能取出下一个，所以真正的执行时间其实受多个因素影响，1个是前面消息执行的时长，2是如果遇到同步屏障把同步消息阻塞住了，那同步消息执行时间就必须还得依赖同步屏障移除时间。所以，postDelayed只是个相对性的时间并不能达到准确。

最后还有一个小细节：

```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
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
从Handler的处理消息方法可以看出，处理消息时优先判断是否有callback。所以如果创建Handler的时候指定了Callback，那处理消息的时候执行的是Callback的handleMessage

以上是我对Handler的全部理解，最后附上一篇Handler的面试问题供日后翻阅：
[Handler面试题汇总](https://www.jianshu.com/p/3063c4ab40bd)

   