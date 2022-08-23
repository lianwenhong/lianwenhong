---
title: Jetpack-Lifecycle
date: 2022-08-23 15:56:23
tags:
categories: Android
---
## Lifecycle
Lifecycle库是Google在2017年IO大会上发布的一个库。本质上是通过观察者模式来帮助开发者实现对Activity、Fragment、Service、Application(应用)声明周期的监测。

## Lifecycle的依赖

**gradle :app:dependencies可查看项目模块依赖树**

如果你的应用已经完成了Support到Androidx的迁移，那么只需要依赖

```
implementation 'androidx.appcompat:appcompat:$version' // 具体版本可查看官网
```
通过查看appcompat的依赖关系可以看出，appcompat内部已经依赖了
```
androidx:lifecycle:lifecycle-runtime:$version
```
该库内部又依赖了
```
androidx:lifecycle:lifecycle-common:$version
```
`lifecycle-common`是lifecycle的基础库，相当于lifecycle的底层服务库。例如`Lifecycle`、`LifecycleRegistry`、`LifecycleObserver`、`LifecycleOwner`这些基础类都是在`lifecycle-common`库中。

Google将lifecycle相关功能进行了拆分，开发者可以按需引入以减小应用的包体积，截止目前最新版本已经更新到2.6.0-alpha01
```
dependencies {
    def lifecycle_version = "2.6.0-alpha01"
    def arch_version = "2.1.0"

    // ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycle_version"
    // ViewModel utilities for Compose
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:$lifecycle_version"
    // LiveData
    implementation "androidx.lifecycle:lifecycle-livedata-ktx:$lifecycle_version"
    // Lifecycles only (without ViewModel or LiveData)
    implementation "androidx.lifecycle:lifecycle-runtime-ktx:$lifecycle_version"

    // Saved state module for ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:$lifecycle_version"

    // Annotation processor
    kapt "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
    // alternately - if using Java8, use the following instead of lifecycle-compiler
    implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

    // optional - helpers for implementing LifecycleOwner in a Service
    implementation "androidx.lifecycle:lifecycle-service:$lifecycle_version"

    // optional - ProcessLifecycleOwner provides a lifecycle for the whole application process
    implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"

    // optional - ReactiveStreams support for LiveData
    implementation "androidx.lifecycle:lifecycle-reactivestreams-ktx:$lifecycle_version"

    // optional - Test helpers for LiveData
    testImplementation "androidx.arch.core:core-testing:$arch_version"

    // optional - Test helpers for Lifecycle runtime
    testImplementation "androidx.lifecycle:lifecycle-runtime-testing:$lifecycle_version"
}

```

## Lifecycle设计框架
lifecycle的设计其实非常简单，就是一个典型的观察者模式。假设让我们用自己的思路实现一个最简单的监测Activity生命周期的观察者，动手之前思考一下观察者模式必备的几个角色：**1.被观察者**，**2.观察者**，**3.调度类（管理观察者和被观察者之间关系）**。那我们实现一下：

```
/**
 * 被观察者
 * 也就是说发生变化的那个角色
 * 比如我们现在想实现的是监测Activity生命周期，那Activity就扮演这个角色
 * 而它的变化就是代码走到了哪个生命周期回调，比如启动Activity时走了onCreate是一种变化，页面渲染完成走了onResume又是一种变化
 */
class LifeActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 首先给管理类添加一个观察者
        LifeManager.INSTANCE.addObserver(LifeObserver())
        // 被观察者走到了onCreate，此时我们应该通知给观察者
        LifeManager.INSTANCE.notifyOnCreate()
    }

    override fun onResume() {
        super.onResume()
        LifeManager.INSTANCE.notifyOnResume()
    }
}

/**
 * 观察者
 * 也就是响应变化的类
 * 主要就是当被观察者的变化发生时，我们想要做什么事就在这个角色中实现
 * 我们现在需求时观察到Activity如果走到onCreate和onResume就打印日志
 */
class LifeObserver {
    fun onCreate() {
        LogUtils.d(" LifeObserver --> onCreate")
    }

    fun onResume() {
        LogUtils.d(" LifeObserver --> onResume")
    }
}

/**
 * 单例
 * 观察者模式的调度类，主要实现的功能有：
 * 1.观察者的注册、解除管理
 * 2.当被观察者发生变化时通知观察者，也就是将观察者和被观察者联系在一起
 */
class LifeManager private constructor() {
    companion object {
        val INSTANCE: LifeManager by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
            LifeManager();
        }
    }

    // 本地维护一个观察者列表，以便当被观察者发生变化时可以通知到所有观察者
    private val lifeObservers = mutableListOf<LifeObserver>()

    // 添加一个观察者
    fun addObserver(observer: LifeObserver) {
        if (!lifeObservers.contains(observer)) {
            lifeObservers.add(observer)
        }
    }

    fun removeObserver(observer: LifeObserver) {
        if (lifeObservers.contains(observer)) {
            lifeObservers.remove(observer)
        }
    }

    // 被观察者走到了onCreate，此时需要通知所有观察者让它们知道
    fun notifyOnCreate() {
        for (o in lifeObservers) {
            o.onCreate()
        }
    }

    // 被观察者走到了onResume，此时需要通知所有观察者让它们知道
    fun notifyOnResume() {
        for (o in lifeObservers) {
            o.onResume()
        }
    }
}
```
这是一个最基础最简单的观察者模式，不考虑多线程，没有任何封装。但是足以体现观察者模式的思路，而Lifecycle正是使用这种方式来进行生命周期管理的。

将Lifecycle库中的角色与本例中的角色一一对应就是这样的：

```
LifecycleOwner --> LifeActivity //LifecycleOwner只是做了一个抽象封装，我们后面再详细说

LifecycleObserver --> LifeObserver

LifecycleRegistry --> LifeManager
```
#### LifecycleOwner

```
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```
这个类对应的角色是被观察者，Google将它抽象出来并开放了一个getLifecycle()方法为的是让用户在使用时无需再去关心如何获取生命周期管理类，你可以发现我们以Activity为例当我们想要获得一个带有生命周期管理功能的Activity页面那建议是继承`AppCompatActivity`类，其最终有个父类是`androidx.core.app.ComponentActivity`:

```
public class ComponentActivity extends Activity implements
        LifecycleOwner,
        KeyEventDispatcher.Component {
   
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    
    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
    
```
而`LifecycleRegistry`对应的不就是管理类吗，所以设计`LifecycleOwner`只是为了让开发者在使用Activity时内部可以更轻松正确地去获得管理类，避免每次自己去创建产生不必要的麻烦。
我们也可以将自己实现的那个小demo做一个简单的改变，抽象出一个自己的`BaseActivity`，后续子类继承它就可以实现这种更规范的代码：

```
interface LifeObject {
    var lifeManager: LifeManager
}

open class BaseActivity : Activity(), LifeObject {
    override var lifeManager: LifeManager
        get() = LifeManager.INSTANCE
        set(value) {}
}

class LifeActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifeManager.notifyOnResume()
    }

    override fun onResume() {
        super.onResume()
        lifeManager.notifyOnResume()
    }
}
```
`LifecycleOwner`很清楚了。

#### LifecycleObserver

```
public interface LifecycleObserver {

}
```
没有任何代码，其实这也是一个小抽象。系统已经为我们实现了几个默认的Activity相关的默认观察者，比如：
```
public interface DefaultLifecycleObserver extends FullLifecycleObserver {
    @Override
    default void onCreate(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStart(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onResume(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onPause(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStop(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onDestroy(@NonNull LifecycleOwner owner) {
    }
}
```
很简单，就是写了一堆生命周期相应的回调方法。因为它所提供的是一种能力，所以只是把方法封装给开发者，开发者收到监听时想做的事情都是五花八门的，所以直接通过继承就可以实现各自想要的炫酷效果。

还有一个默认实现是：

```
public interface LifecycleEventObserver extends LifecycleObserver {
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}

```
这个更加简单一点，系统给生命周期对应的方法定义了一一对应的枚举值，每次走到某个生命周期只需调用这个统一的onStateChanged回调并将相应的枚举传递过去开发者即可自行实现想要的逻辑。

```
public class LifecycleRegistry extends Lifecycle {

    // 观察者集合
    private FastSafeIterableMap<LifecycleObserver, LifecycleRegistry.ObserverWithState> mObserverMap;

    // 这里将LifecycleOwner传递过来只是为了在通知观察者的时候把被观察者对象也传递过去方便开发者在需要使用到被观察者时能精准地获取到，所以充分体现了设计框架的时候要考虑到各种使用场景以便给使用者更好的使用体验
    private final WeakReference<LifecycleOwner> mLifecycleOwner;
    
    private LifecycleRegistry(@NonNull LifecycleOwner provider, boolean enforceMainThread) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
        // 这里其实是判断是否线程安全，系统默认是必须在主线程中调用这样就避免了线程安全问题，其实也是有道理的因为生命周期肯定都是在主线程中调用的
        mEnforceMainThread = enforceMainThread;
    }
    
    // 事件分发，扮演的角色其实类似于我们demo中的notifyOnCreate、notifyOnResume方法，只是这里它做了一系列的状态比较等更严谨的操作
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent");
        moveToState(event.getTargetState());
    }
    
    public void addObserver(@NonNull LifecycleObserver observer) {
        this.enforceMainThreadIfNeeded("addObserver");
        State initialState = this.mState == State.DESTROYED ? State.DESTROYED : State.INITIALIZED;
        LifecycleRegistry.ObserverWithState statefulObserver = new LifecycleRegistry.ObserverWithState(observer, initialState);
        // 将观察者加入观察者集合
        LifecycleRegistry.ObserverWithState previous = (LifecycleRegistry.ObserverWithState)this.mObserverMap.putIfAbsent(observer, statefulObserver);
        if (previous == null) {
            LifecycleOwner lifecycleOwner = (LifecycleOwner)this.mLifecycleOwner.get();
            if (lifecycleOwner != null) {
                boolean isReentrance = this.mAddingObserverCounter != 0 || this.mHandlingEvent;
                // 计算出当前Activity处于哪个生命周期
                State targetState = this.calculateTargetState(observer);
                ++this.mAddingObserverCounter;
                
                // while循环是为了将添加观察者之前所经历过的生命周期都按顺序回调一遍
                while(statefulObserver.mState.compareTo(targetState) < 0 && this.mObserverMap.contains(observer)) {
                    this.pushParentState(statefulObserver.mState);
                    Event event = Event.upFrom(statefulObserver.mState);
                    if (event == null) {
                        throw new IllegalStateException("no event up from " + statefulObserver.mState);
                    }
                    
                    // 给观察者回调事件
                    statefulObserver.dispatchEvent(lifecycleOwner, event);
                    this.popParentState();
                    targetState = this.calculateTargetState(observer);
                }

                if (!isReentrance) {
                    this.sync();
                }

                --this.mAddingObserverCounter;
            }
        }
    }
    
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;
    
        ObserverWithState(LifecycleObserver observer, State initialState) {
            // 主要是2类，一类是LifecycleEventObserver另一类是FullLifecycleObserver相关的观察者
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }
    
        // 分发生命周期对应方法
        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = event.getTargetState();
            mState = min(mState, newState);
            // 在mLifecycleObserver中做中转
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}

```
而在Activity走到各个生命周期时会回调给`LifecycleRegistry->handleLifecycleEvent`方法，而系统为我们做这个逻辑的代码是在FragmentActivity中，我们实现继承的AppCompatActivity是继承FragmentActivity的

```
public class FragmentActivity extends ComponentActivity implements
        ActivityCompat.OnRequestPermissionsResultCallback,
        ActivityCompat.RequestPermissionsRequestCodeValidator {
        ...
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    
        mFragmentLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        mFragments.dispatchCreate();
    }
    
    // resume事件监听相对绕一点，可以从ActivityThread中开始走Activity的resume流程最终调用到Activity类中performResume然后会回调到这个onPostResume()方法中
    protected void onResumeFragments() {
        mFragmentLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
        mFragments.dispatchResume();
    }
            
}
```
所以至此Activity生命周期监听的大概框架设计原理已经了解完了，归根结底很简单就是一个观察者模式。



## Lifecycle常见使用场景及其实现思路解析
### 监测应用前后台切换
我们经常要判断应用是否处于前后台，自从有了Lifecycle之后这个需求就很简单，不再需要去getRunningTask...。
之前说过Lifecycle可以按需引入。我们现在引入针对整个应用的Lifecycle

```
implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version" //版本建议使用当前最新的，可看官方文档
```
引入之后我们的Application可以这么写：

```
class JApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // 通过lifecycle监测应用切换前后台操作
        ProcessLifecycleOwner.get().lifecycle.addObserver(AppLifecycleObserver())
        registerLifecycle()
    }
}

class AppLifecycleObserver : LifecycleEventObserver {
    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        when (event) {
            Lifecycle.Event.ON_RESUME -> {
                LogUtils.d("当前应用处于前台")
            }
            Lifecycle.Event.ON_STOP -> {
                LogUtils.d("当前应用处于后台")
            }
            Lifecycle.Event.ON_CREATE,
            Lifecycle.Event.ON_START,
            Lifecycle.Event.ON_PAUSE,
            Lifecycle.Event.ON_DESTROY,
            Lifecycle.Event.ON_ANY
            -> {
            }
        }
    }
}
```
我们当应用处于前台时一定会走到观察者的onStart、onResume，当应用处于后台时一定会走onPause、onStop。所以我们只需要观察onResume和onStop这2个节点的执行情况即可。

我们在JApplication的onCreate()中通过ProcessLifecycleOwner.get()获取到一个ProcessLifecycleOwner对象。然后调用其内部的Lifecycle的addObserver来实现对应用生命周期的观察，走进源码可以看到熟悉的`private final LifecycleRegistry mRegistry = new LifecycleRegistry(this);`，在前面讲原理的时候已经知道了LifecycleRegistry中要分发事件给观察者的话都是通过其内部的handleLifecycleEvent()来实现的，所以我们看下应用是在什么场景下调用了handleLifecycleEvent()：
```
public class ProcessLifecycleOwner implements LifecycleOwner {

    ...
    
    private final LifecycleRegistry mRegistry = new LifecycleRegistry(this);

    private Runnable mDelayedPauseRunnable = new Runnable() {
        @Override
        public void run() {
            dispatchPauseIfNeeded();
            dispatchStopIfNeeded();
        }
    };

    ActivityInitializationListener mInitializationListener =
            new ActivityInitializationListener() {
                @Override
                public void onCreate() {
                }

                @Override
                public void onStart() {
                    activityStarted();
                }

                @Override
                public void onResume() {
                    activityResumed();
                }
            };

    private static final ProcessLifecycleOwner sInstance = new ProcessLifecycleOwner();

    @NonNull
    public static LifecycleOwner get() {
        return sInstance;
    }

    static void init(Context context) {
        sInstance.attach(context);
    }

    void activityStarted() {
        mStartedCounter++;
        if (mStartedCounter == 1 && mStopSent) {
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
            mStopSent = false;
        }
    }

    void activityResumed() {
        mResumedCounter++;
        if (mResumedCounter == 1) {
            if (mPauseSent) {
                mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
                mPauseSent = false;
            } else {
                mHandler.removeCallbacks(mDelayedPauseRunnable);
            }
        }
    }

    void activityPaused() {
        mResumedCounter--;
        if (mResumedCounter == 0) {
            mHandler.postDelayed(mDelayedPauseRunnable, TIMEOUT_MS);
        }
    }

    void activityStopped() {
        mStartedCounter--;
        dispatchStopIfNeeded();
    }

    void dispatchPauseIfNeeded() {
        if (mResumedCounter == 0) {
            mPauseSent = true;
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
        }
    }

    void dispatchStopIfNeeded() {
        if (mStartedCounter == 0 && mPauseSent) {
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
            mStopSent = true;
        }
    }

    private ProcessLifecycleOwner() {
    }

    @SuppressWarnings("deprecation")
    void attach(Context context) {
        ...
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @RequiresApi(29)
            @Override
            public void onActivityPreCreated(@NonNull Activity activity,
                    @Nullable Bundle savedInstanceState) {
                
                Api29Impl.registerActivityLifecycleCallbacks(activity,
                        new EmptyActivityLifecycleCallbacks() {
                            @Override
                            public void onActivityPostStarted(@NonNull Activity activity) {
                                //  onStart回调
                                activityStarted();
                            }

                            @Override
                            public void onActivityPostResumed(@NonNull Activity activity) {
                                //  onResume回调
                                activityResumed();
                            }
                        });
            }

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                if (Build.VERSION.SDK_INT < 29) {
                    // 这是一个透明Fragment，后续有机会再详细撸 ReportFragment.get(activity).setProcessListener(mInitializationListener);
                }
            }

            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }

            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mRegistry;
    }

    @RequiresApi(29)
    static class Api29Impl {
        private Api29Impl() {
            // This class is not instantiable.
        }

        @DoNotInline
        static void registerActivityLifecycleCallbacks(@NonNull Activity activity,
                @NonNull Application.ActivityLifecycleCallbacks callback) {
            activity.registerActivityLifecycleCallbacks(callback);
        }
    }
}
```
很清楚的看到ProcessLifecycleOwner中有好几个activityXXX()这样的生命周期相关的回调方法最终调用了`mRegistry.handleLifecycleEvent`，以activityStarted为例:
```
void activityStarted() {
    mStartedCounter++;
    if (mStartedCounter == 1 && mStopSent) {
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
        mStopSent = false;
    }
}
```
所以至此，监测应用生命周期共有这几种回调值

```
ON_CREATE、ON_START、ON_RESUME、ON_PAUSE、ON_STOP
```
现在我们只需要知道是哪里调用了这几个生命周期回调方法就等于把整个应用的生命周期监控了解清楚了

通过跟踪调用栈我们最终找到`ProcessLifecycleOwner.attach()`方法中回调了ON_CREATE事件，然后注册了registerActivityLifecycleCallbacks()。再往上追踪就涉及到ProcessLifecycleOwner的构建过程。

我们从这个对象其实早在应用刚启动时进行过初始化了，是在Androidx的startup-runtime中完成的：
{% asset_img startup-runtime.png startup-runtime %}
```
public final class ProcessLifecycleInitializer implements Initializer<LifecycleOwner> {

    @NonNull
    @Override
    public LifecycleOwner create(@NonNull Context context) {
        ...
        LifecycleDispatcher.init(context);
        ProcessLifecycleOwner.init(context);
        return ProcessLifecycleOwner.get();
    }

    @NonNull
    @Override
    public List<Class<? extends Initializer<?>>> dependencies() {
        return Collections.emptyList();
    }
}
```
`ProcessLifecycleOwner.init(context)`执行了ProcessLifecycleOwner的初始化，最终走到ProcessLifecycleOwner.attach()中
在attach()中会向Application对象注册ActivityLifecycleCallbacks对象
```
public class Application extends ContextWrapper implements ComponentCallbacks2 {
    ...
    public void registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
        synchronized (mActivityLifecycleCallbacks) {
            mActivityLifecycleCallbacks.add(callback);
        }
    }
    
    public void unregisterActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
        synchronized (mActivityLifecycleCallbacks) {
            mActivityLifecycleCallbacks.remove(callback);
        }
    }
    
    private Object[] collectActivityLifecycleCallbacks() {
        Object[] callbacks = null;
        synchronized (mActivityLifecycleCallbacks) {
            if (mActivityLifecycleCallbacks.size() > 0) {
                callbacks = mActivityLifecycleCallbacks.toArray();
            }
        }
        return callbacks;
    }
    
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreCreated(@NonNull Activity activity,
            @Nullable Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreCreated(activity,
                        savedInstanceState);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityCreated(@NonNull Activity activity,
            @Nullable Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityCreated(activity,
                        savedInstanceState);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostCreated(@NonNull Activity activity,
            @Nullable Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostCreated(activity,
                        savedInstanceState);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreStarted(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreStarted(activity);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityStarted(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityStarted(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostStarted(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostStarted(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreResumed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreResumed(activity);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityResumed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityResumed(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostResumed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostResumed(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPrePaused(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPrePaused(activity);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityPaused(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityPaused(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostPaused(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostPaused(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreStopped(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreStopped(activity);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityStopped(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityStopped(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostStopped(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostStopped(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreSaveInstanceState(@NonNull Activity activity,
            @NonNull Bundle outState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreSaveInstanceState(
                        activity, outState);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivitySaveInstanceState(@NonNull Activity activity,
            @NonNull Bundle outState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivitySaveInstanceState(activity,
                        outState);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostSaveInstanceState(@NonNull Activity activity,
            @NonNull Bundle outState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostSaveInstanceState(
                        activity, outState);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPreDestroyed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPreDestroyed(activity);
            }
        }
    }

    @UnsupportedAppUsage
    /* package */ void dispatchActivityDestroyed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityDestroyed(activity);
            }
        }
    }

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        /* package */ void dispatchActivityPostDestroyed(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityPostDestroyed(activity);
            }
        }
    }

    /* package */ void dispatchActivityConfigurationChanged(@NonNull Activity activity) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((ActivityLifecycleCallbacks) callbacks[i]).onActivityConfigurationChanged(
                        activity);
            }
        }
    }
    ...
}
```
在Application中又有一系列以dispatchActivityXXX为方法名的方法来通知所有注册进来的观察者，而什么时候调用呢，其实是在每一个Activity中调用的，以ON_CREATE为例：

```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {
        ...
        
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    ...
    dispatchActivityPostCreated(icicle);
    ...
}
    
    private void dispatchActivityPostCreated(@Nullable Bundle savedInstanceState) {
    Object[] callbacks = collectActivityLifecycleCallbacks();
    if (callbacks != null) {
        for (int i = 0; i < callbacks.length; i++) {
            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostCreated(this,
                    savedInstanceState);
        }
    }
    getApplication().dispatchActivityPostCreated(this, savedInstanceState);
}
        
}

```
熟悉Activity启动流程的应该对这一块不会陌生吧

可以看出来，每个Activity执行过程中都会回调对应的生命周期给Application，然后Application再通过观察者模式分发给每一个观察者，这样我们就收到了回调。


### 管理应用页面情况（维护应用Activity栈）
经常我们一个应用开发时都会写一个管理类来维护一份自己的Activity栈，早前的做法是在BaseActivity中的onCreate和onDestroy做对应的入栈出栈。现在不需要那么做了，直接在Application中通过`registerActivityLifecycleCallbacks`就可以实现入栈出栈，原理我们在上面已经讲过了：**观察者模式**

所以现在完整的JApplication代码如下

```
class JApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // 通过lifecycle监测应用切换前后台操作
        ProcessLifecycleOwner.get().lifecycle.addObserver(AppLifecycleObserver())
        registerLifecycle()
    }

    fun getApp(): Application {
        return this
    }

    /**
     * 管理应用的页面（Activity）
     */
    private fun registerLifecycle() {
        registerActivityLifecycleCallbacks(object : ActivityLifecycleCallbacks {
            override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
                AppManager.addActivity(activity)
            }

            override fun onActivityStarted(activity: Activity) {}
            override fun onActivityResumed(activity: Activity) {
//                AppManager.getInstance().setCurrentActivity(activity)
            }

            override fun onActivityPaused(activity: Activity) {}
            override fun onActivityStopped(activity: Activity) {}
            override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
            override fun onActivityDestroyed(activity: Activity) {
                AppManager.killActivity(activity)
            }
        })
    }
}
```
### 监控Activity生命周期
监控Activity生命周期也很简单，继承有Lifecycle功能的Activity（如AppCompactActivity）然后获取其内部的Lifecycle，执行`addObserver()`，一切搞定。为了让代码更加优雅一点我封装了一个BaseActivity，并且未来准备搭配ViewModel使用所以又封装了一个ActivityViewModel

```
/**
 *  抽象针对Activity页面的ViewModel
 *
 *  ViewModel拥有自己独立的生命周期，可保证Activity被重建时数据不会丢失
 *
 *  可在此ViewModel中实现一些基础功能，例如生命周期监测
 */
open class ActivityViewModel : ViewModel(), DefaultLifecycleObserver {

    override fun onCreate(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onCreate")
    }

    override fun onStart(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onStart")
    }

    override fun onResume(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onResume")
    }

    override fun onPause(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onPause")
    }

    override fun onStop(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onStop")
    }

    override fun onDestroy(owner: LifecycleOwner) {
        LogUtils.d("${owner.javaClass.simpleName} invoke onDestroy")
    }

}

abstract class BaseActivity<T : ActivityViewModel> : ComponentActivity() {
    var viewModel: T? = null
        get() = field

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel = bindViewModel()
        viewModel?.let { lifecycle.addObserver(it) }
    }

    abstract fun bindViewModel(): T?
}
```
如此一来每个Activity页面只需要继承BaseActivity并提供页面对应的ActivityViewModel子类即可实现生命周期监测的能力。

### 监控Fragment生命周期
Fragment中使用Lifecycle和Activity很相似，内部实现原理也是Fragment中提供一个Lifecycle实现类供开发者插入观察者，然后FragmentStateManager会在Fragment处于不同的生命周期调用对应的生命周期回调，开发者只需要在Fragment中addObserver即可。不详细说了

### 监控Service生命周期
Service中使用Lifecycle监控Service的生命周期更加简单，只需继承`LifecycleService`类然后在Service中写入`lifecycle.addObserver(ServiceLifecycleObserver())`即可实现，原理特别简单相信大家一看就看得懂。

本质上是系统提供一个`ServiceLifecycleDispatcher`类，该类内部会返回一个Lifecycle供开发者插入观察者等功能。然后又提供一系列生命周期相关的调用方法，`LifecycleService`做了个简单的封装在onCreate等声明周期方法中调用了`mDispatcher.onServicePreSuperOnCreate();`这样来通知观察者，我又简单抽象了一个基类来适配Android8.0应用处于后台时无法启动Service的兼容逻辑：
```
/**
 * 后来发现这个类实际上Android系统已经提供了一个一模一样的类：LifecycleService
 * 所以此处将生命周期相关代码注释并且直接继承LifecycleService
 */
abstract class JService : LifecycleService() /*Service(), LifecycleOwner*/ {

    fun startForegroundNotification(channelId: String, notificationId: Int) {
        if (SystemUtils.isOverO8()) {
            val channel = NotificationChannel(
                packageName,
                channelId,
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NOTIFICATION_SERVICE)?.let { it as NotificationManager }
            manager?.createNotificationChannel(channel)
            val notification = Notification.Builder(
                this,
                packageName
            ).build()
            startForeground(notificationId, notification)
        }
    }

//    private val mDispatcher: ServiceLifecycleDispatcher = ServiceLifecycleDispatcher(this)
//
//    override fun getLifecycle(): Lifecycle {
//        return mDispatcher.lifecycle
//    }
//
//    fun onServicePreSuperOnCreate() {
//        mDispatcher.onServicePreSuperOnCreate()
//    }
//
//    fun onServicePreSuperOnStart() {
//        mDispatcher.onServicePreSuperOnStart()
//    }
//
//    fun onServicePreSuperOnBind() {
//        mDispatcher.onServicePreSuperOnBind()
//    }
//
//    fun onServicePreSuperOnDestroy() {
//        mDispatcher.onServicePreSuperOnDestroy()
//    }
}
```



