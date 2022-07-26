---
title: Android小知识-HandlerThread
date: 2022-07-26 17:11:15
tags: 
categories: Android
---
### 线程间通信常规操作
Android线程间通信主要方式是使用Handler相信没有一个做Android的人不知道。
如果对Handler还不了解可以回顾一下我以前的文章：[Android中多线程通信：Handler的理解](http://lianwenhong.top/2022/03/25/Android中多线程通信：Handler的理解/)

注： 后续称HandlerThread创建的线程为**HandlerThread线程**，调用HandlerThread.getLooper()的线程成为**调用线程**

### HandlerThread用法
如果你在开发android项目时想实现一个带消息队列的子线程，那我建议你不要重复造轮子，你可以看看HandlerThread这个类，它就是系统封装好的一个很好用的带消息队列的子线程。
HandlerThread能解决以下几件事情：
1. 实现带消息队列的子线程，可用它来做耗时操作。
2. 可实现线程间通信。
3. 可实现线程的线程的复用避免了线程频繁创建销毁带来的开销。

**简单概括就是HandlerThread继承自Thread，在HandlerThread这个线程启动的时候其内部run()方法中创建并初始化了该线程的Looper，并且在初始化Looper和使用Looper的阶段通过synchronized同步锁的方式保证了Looper的准确性。该类通过系统的消息轮训器Looper来实现消息队列以及实现线程的复用，其内部也提供了关闭子线程的方法以便开发者可以在自己想要的场景下开启子线程或者关闭子线程。**

其实HandlerThread只是系统对子线程开发需求的一个小封装而已，但是我个人觉得挺实用的。先来看它的使用方式：

```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private HandlerThread handlerThread;
    Handler handler;
    int count;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button start = findViewById(R.id.id_btn_begin);
        start.setOnClickListener(this);
        Button send = findViewById(R.id.id_btn_send);
        send.setOnClickListener(this);
        Button finish = findViewById(R.id.id_btn_finish);
        finish.setOnClickListener(this);
    }

    // 创建HandlerThread子线程和Handler消息处理器
    public void startThread() {
        count = 0;
        handlerThread = new HandlerThread("my-thread") {
            @Override
            protected void onLooperPrepared() {
                super.onLooperPrepared();
                Log.e("lianwenhong", " >>> onLooperPrepared <<< ");
            }
        };
        handlerThread.start();// 必须先执行start()之后再来执行handlerThread.getLooper();，否则得到的Looper对象永远是null
        // 创建handler时传入HandlerThread的Looper对象，后续通过这个handler对象发送的消息都会被放在handlerThread线程中执行
        handler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                    case 1:
                        Log.e("lianwenhong", " >>> handle 1");
                        break;
                    case 2:
                        Log.e("lianwenhong", " >>> handle 2");
                        break;
                    default:
                        Log.e("lianwenhong", " >>> handle default");
                }
            }
        };
    }

    // 当有需要在子线程中处理的逻辑就通过消息发送给子线程，此时消息会进入HanderThread线程中的消息队列
    public void sendMessage() {
        Message msg1 = handler.obtainMessage(++count);
        handler.sendMessage(msg1);
    }

    public void finishThread() {
        handlerThread.quit();
    }


    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.id_btn_begin:
                startThread();
                break;
            case R.id.id_btn_send:
                sendMessage();
                break;
            case R.id.id_btn_finish:
                finishThread();
                break;
        }
    }
}
```
以上是HanderThread的使用方式：
1. 创建一个HandlerThread对象，并且开启它。
2. 创建一个Handler，给Handler传入HandlerThread的Looper对象，复写handleMessage方法
3. 向2中的Handler对象发送Message时，该消息对应的事件会在HandlerThread线程中执行。

### HandlerThread原理
handlerThread原理不难，总的代码也才100多行，咱们来撸一遍：

```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable
    Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    /**
     * 没什么特别，只是设置了一下线程的优先级为默认优先级
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }

    /**
     * 这是一个回调表示该线程的Looper已经准备就绪，此时开发者可以在收到这个回调时做一些初始化工作
     */
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        // 此处使用synchronized加锁是确保调用线程执行handlerThread.getLooper()获取mLooper对象时mLooper已经被正确赋值了
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

    /**
     * 返回handlerThread所关联的Looper，如果handlerThread线程已经死亡则返回null
     * 如果handlerThread线程未执行过start()方法的话就会阻塞调用线程
     *
     * @return The looper.
     */
    public Looper getLooper() {
        // 如果handlerThread线程未启动或已经死亡，则返回null
        if (!isAlive()) {
            return null;
        }

        boolean wasInterrupted = false;

        // If the thread has been started, wait until the looper has been created.
        // 获取Looper时首先要确保线程已经启动并且mLooper已经赋值成功，否则将会返回null导致传递给Handler时会抛出java.lang.NullPointerException:
        // Attempt to read from field 'android.os.MessageQueue android.os.Looper.mQueue' on a null object reference异常
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    // 调用线程与handlerThread并不处于同一个线程，所以并不能保证当执行handlerThread.getLooper时handlerThread线程中mLooper已经赋值完毕。
                    // 所以如果mLooper还未赋值成功时就执行wait()阻塞调用线程并释放锁对象，直到handlerThread线程获取到执行权并且对mLooper赋值完成之后handlerThread线程
                    // 会通过notifyAll()来通知正在wait()的调用线程，此时调用线程就可以得到正确的mLooper对象而不是空
                    wait();
                } catch (InterruptedException e) {
                    wasInterrupted = true;
                }
            }
        }

        /*
         * 如果线程还未来得及被notify就收到了中断请求则直接中断调用线程
         */
        if (wasInterrupted) {
            Thread.currentThread().interrupt();
        }

        return mLooper;
    }

    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    /**
     * 退出Looper消息循环
     * 如果队列中还有未执行的消息，那么不管是延时还是非延时的消息统一抛弃并停止Looper消息循环
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    /**
     * 退出Looper消息循环
     * 如果队列中还有延时任务未执行则将延时任务全部清除，也就是说延时任务直接被抛弃得不到执行
     * 如果队列中还有非延时任务未执行则会一直等到所有非延时任务都得到执行之后才真正停止Looper消息循环
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    /**
     * 返回线程id
     */
    public int getThreadId() {
        return mTid;
    }
}
```
上面代码中的注释已经很详细的说明了HandlerThread的原理，结合它的使用方式看源码应该是一目了然了。
#### 有几个点需要强调的：
1. 必须先调用HandlerThread.start()再去获取HandlerThread线程所关联的Looper对象(getLooper())，否则在未调用HandlerThread.start()之前调用HandlerThread.getLooper()得到的永远是null
2. 从run()和getLooper()方法中的synchronized逻辑可以看出来，调用线程在执行HandlerThread.getLooper()时有可能会发生阻塞，不过这个阻塞时长应该很短，只需了解这一点就行。阻塞是因为如果是HandlerThread线程run()方法还没执行到synchronized代码块时HandlerThread线程就失去cpu执行权那么假设此时cpu执行权被调用线程拿到并且执行到HandlerThread.getLooper()中的同步代码块，那么调用线程将被阻塞知道run()中同步代码块执行完毕。
3. 因为Looper消息队列是串行一条一条执行的，所以太过耗时的消息势必会影响后续的消息处理（假设加入队列的消息有A、B两个，A消息历时10s才完成那么B消息得到执行必然是在10s之后）。这点在业务开发中要自行斟酌，例如网络请求是一种不可预测的大耗时操作，不建议放在HandlerThread中来执行，而简单的文件读写一般是毫秒级的，可以放在HandlerThraed中。其实不光是HandlerThread有这个问题，任何消息单任务执行的队列都有这个问题。
