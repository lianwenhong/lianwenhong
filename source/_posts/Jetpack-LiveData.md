---
title: Jetpack-LiveData
date: 2022-09-15 10:47:11
tags:
categories: Android
---
## LiveData简介
[LiveData](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData)是一种可观察的数据存储器类。与常规的可观察类不同，LiveData具有生命周期感知能力，意指它遵循其他应用组件（如activity、fragment或service）的生命周期。这种感知能力可确保LiveData仅更新处于活跃生命周期状态的应用组件观察者。

如果观察者（由 Observer类表示）的生命周期**处于STARTED或RESUMED状态**，则LiveData会认为该观察者处于活跃状态。LiveData只会将更新通知给活跃的观察者。为观察LiveData对象而注册的非活跃观察者不会收到更改通知。

您可以注册与实现LifecycleOwner接口的对象配对的观察者。有了这种关系，当相应的Lifecycle对象的状态变为DESTROYED时，便可移除此观察者。这对于activity和fragment特别有用，因为它们可以放心地观察 LiveData 对象，而不必担心内存泄露（当 activity 和 fragment 的生命周期被销毁时，系统会立即退订它们）。

LiveData一般声明在ViewModel中搭配ViewModel一起使用。

## 使用 LiveData 的优势
1. **确保界面符合数据状态**
> LiveData 遵循观察者模式。当底层数据发生变化时，LiveData会通知 Observer对象。您可以整合代码以在这些Observer对象中更新界面。这样一来，您无需在每次应用数据发生变化时更新界面，因为观察者会替您完成更新。
2. **不会发生内存泄漏**
> 观察者会绑定到Lifecycle对象，并在其关联的生命周期遭到销毁后进行自我清理。
3. **不会因 Activity 停止而导致崩溃**
> 如果观察者的生命周期处于非活跃状态（如返回堆栈中的activity），它便不会接收任何LiveData事件。
4. **不再需要手动处理生命周期**
> 界面组件只是观察相关数据，不会停止或恢复观察。LiveData会在内部自动管理所有这些操作，因为它在观察时可以通过LifecycleOwner感知相关的生命周期状态变化。
5. **数据始终保持最新状态**
> 如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据，并且如果在非活跃状态时数据经过了多次变化也只会收到最新的那次的通知。例如，曾经在后台的Activity会在返回前台后立即接收最新的数据。
6. **适当的配置更改**
> 如果由于配置更改（如设备旋转）而重新创建了activity或fragment，它会立即接收最新的可用数据。当然这也归功于其LiveData和ViewModel的搭配使用，这样才能是的LiveData在配置更改后依然保留在内存中。
7. **共享资源**
> 您可以使用单例模式扩展LiveData对象以封装系统服务，以便在应用中共享它们。LiveData对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察LiveData对象。如需了解详情，请参阅[扩展LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata.html#extend_livedata)。

## LiveData的简单使用
```
class MainViewModel : ViewModel() {
    val userName: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }
}

class MainActivity : AppCompatActivity() {

    val viewModel: ViewModel by lazy {
        ViewModelProvider(this).get(MainViewModel::class.java)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        initView()
    }
    
    private fun initView() {

        txtName = findViewById(R.id.txt_name)
        btnChangeInMain = findViewById(R.id.btn_change_in_main)
        btnChangeInOther = findViewById(R.id.btn_change_in_other)
        
        demoLiveData()
    }
    
    private fun demoLiveData() {
        viewModel?.userName?.observe(this) { txtName?.text = "NAME:$it" }
        viewModel?.userName?.observe(this) {
            Toast.makeText(this, "USERNAME改变", Toast.LENGTH_SHORT).show()
        }
        btnChangeInMain?.setOnClickListener {
            viewModel?.userName?.value = "MAIN"
        }

        btnChangeInOther?.setOnClickListener {
            val thread = Thread {
                viewModel?.userName?.postValue("OTHER")
            }
            SystemClock.sleep(2000)
            thread.start()
        }
    }
}
```
上述代码中以最简单的`MutableLiveData`为例，它继承自抽象类`LiveData`。
- 我们在MainViewModel中用MutableLiveData容器存储一个String类型的userName
- 在页面中使用observe()方法传入Observer观察者对象和LifecycleOwner对象使得userName内数据发生变化时或者页面生命周期发生变化时会将userName最新的值回调到Observer的onChanged()方法中。

## LiveData原理解析
讲解LiveData之前最好是先对Lifecycle和ViewModel的实现原理有足够的了解否则对本文阅读可能有点不知所云，特别是对了解《页面生命周期变化时如何将最新的LiveData数据通知给数据观察者以达到更新UI的目的》这部分会比较难理解。

如果不了解可以看我前面2篇文章：[Jetpack-Lifecycle]()，[Jetpack-ViewModel]()

从2个角度来分析LiveData的运行原理：LiveData观察者绑定**LiveData->observe()** 和LiveData数据发生改变 **LiveData->setValue()**

在讲解过程中我们将实现`LifecycleEventObserver`的接口称为生命周期观察者，将`ObserverWrapper`的实现类称为数据观察者，这两者所观察的目标不一样，前者观察页面的生命周期，后者观察LiveData的value变化。

### 1. LiveData->observe()
```
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    // 1.注册过程必须在主线程执行
    assertMainThread("observe");
    // 2.注册过程必须是LifecycleOwner处于活跃状态，也就是页面处于活跃状态
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    // 3.封装一个生命周期感知的观察者包装类LifecycleBoundObserver(把注册进来的observer包装成 一个具有生命周边边界的观察者，其能感知LifecycleOwner的生命周期变化)
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    // 4.判断观察者是否已经存在并且同一个观察者只能感知一个页面的生命周期
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    // 5.将wrapper注册进LifecycleOwner对应的生命周期观察者列表中，这样才最终实现LifecycleBoundObserver对LifecycleOwner的生命周期感知
    owner.getLifecycle().addObserver(wrapper);
}
```
整个注册过程并不复杂
- 先判断当前是否处于主线程，因为其实LiveData主要的作用就是让页面能及时响应数据的变化，而UI改变需要在主线程中执行，所以这一步也是可以理解的
- 判断当前页面是否处于活跃状态，如果此时传入的页面对应的LifecycleOwner生命周期处于非活跃状态则不进行观察，例如如果页面已经被关闭或者处于后台中，那你观察该数据来更新UI对于用户也是无感知的，所以没有意义。如果你非要在后台更新数据，那么你可以直接使用observeForever(@NonNull Observer<? super T> observer)这个方法来实现，后果是你得自己去维护这个LiveData与观察者的解除绑定关系。
- 将LifecycleOwner和Observer封装在一个LifecycleBoundObserver包装类中并保存在LiveData的mObservers中以便后续数据发生变化时能及时响应到各个观察者中，LifecycleBoundObserver可以感知LifecycleOwner的生命周期变化，因为其实现了`LifecycleEventObserver`接口，下面再详解
- 在使用过程中需要注意第4点，同一个Observer数据观察者绑定的LifecycleOwner必须是同一个否则会抛异常。也不难理解，假如同一个数据观察者绑定的页面不一样那么2个页面的生命周期不一样时数据观察者应该响应哪个页面的生命周期就是个问题！
- 最后将该LifecycleBoundObserver包装类注册给LifecycleOwner，这样当页面生命周期发生变化时就会回调到LifecycleBoundObserver->onStateChanged()方法中，在onStateChanged中就会在合适的时机去通知观察者LiveData的数据发生变化了，需要更新UI的请开始吧

LifecycleBoundObserver是LiveData的内部类，LiveData通过这个内部类将观察者（Observer）和页面的生命周期组合在一起使得每一个观察者都对应响应一个页面的生命周期（LifecycleOwner）。
```
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner; // 用来获取页面最新生命周期以及判断数据观察者重新绑定时LifecycleOwner是否是同一个

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    // 当mOwner对应的页面生命周期发生变化时，Lifecycle会回调该方法并传入最新的生命周期状态通知该生命周期观察者
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        // 当页面处于非活跃状态时移除数据观察者，这样能避免没必要的通知提升效率。
        // 这就是为什么在Fragment中使用LiveData时observe中应该传入viewLifecycleOwner而不是传入this，后面再详解
        if (currentState == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            // 页面生命周期变化原因通知数据观察者的入口
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    // 移除生命周期观察者
    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}

void activeStateChanged(boolean newActive) {
    // 如果页面的活跃状态并没有发生变化就直接return，这是一个优化避免不必要的重复通知
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    // 更新一下当前LiveData中有多少个活跃的页面中有数据观察者正在观察它的value，活跃页面数量0-1调用：onActive()，1-0调用：onInactive()。
    // 不重要，好像是有时候用于判断LiveData是否开始正常工作
    changeActiveCounter(mActive ? 1 : -1);
    if (mActive) {
        // 如果页面从非活跃变为活跃状态，则开始处理通知数据观察者的逻辑。
        dispatchingValue(this);
    }
}
```
- LifecycleBoundObserver内部主要工作是监听页面的生命周期变化并在收到生命周期变化回调时及时响应给LiveData的数据观察者。
- 其实LifecycleBoundObserver只关注页面是否存在活跃和非活跃状态之间的切换而不关心具体处于哪个生命周期。这也不难理解，监听页面生命周期无非就是想要在页面从非活跃到活跃状态时能及时的刷新页面上的UI，如果不存在非活跃到活跃状态的切换时并没有必要去刷新页面，例如在页面从STARTED->RESUMED时页面一直处于活跃状态，这时候activeStateChanged()直接执行了return。因为在这过程中首先STARTED中肯定已经通知过数据观察者一次了，即使STARTED->RESUMED过程中LiveData.value又发生过改变就会执行setValue()，setValue()中会通知该LiveData的所有数据观察者，所以在这里也没有必要再重复通知。
- 真正的分发是dispatchingValue(this);注意这里传入的是this，我们放在setValue()流程时再一起解析它

### 2. LiveData->setValue()

```
@MainThread
protected void setValue(T value) {
    // 这是主线程更新值的方式，在子线程中调用就会报错
    assertMainThread("setValue");
    // LiveData数据的每一次改动都会促使这个版本号+1，后续当需要通知给观察者时会使用这个值和每个数据观察者内部的版本做判断，如果mVersion版本比较高才会回调onChanged()
    mVersion++;
    // 赋值，这就是真正
    mData = value;
    // 通知数据观察者，此处传入的是null
    dispatchingValue(null);
}

@SuppressWarnings("WeakerAccess") /* synthetic access */
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        // 如果当前正处于在通知数据观察者阶段，则将其设置为无效
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        // 这是页面生命周期变化触发的通知数据观察者更新
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            // 这是调用setValue()或者postValue()触发的通知数据观察者更新
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

@SuppressWarnings("unchecked")
private void considerNotify(ObserverWrapper observer) {
    // 如果当前数据观察者绑定的页面为非活跃状态，则直接return不通知
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    // 如果在多线程情况下数据观察者绑定的页面生命周期又发生了变化从活跃变为非活跃，那也不更新并且将这个数据观察者内部的活跃状态更新一下
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 如果数据观察者内部的版本号>=当前要更新的数据的版本号那说明数据已经被更新过了，不需要重复更新，return掉
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    // 最终调用了数据观察者的onChanged()方法
    observer.mObserver.onChanged((T) mData);
}
```
其实LiveData.setValue()过程很简单，主要做的就是`mData = value;`给LiveData的数据赋值，然后`observer.mObserver.onChanged((T) mData);`通知给数据观察者说我的值发生了变化。

既然只做了这么点儿事为什么有这么一大堆的代码，还有什么版本判断啊各种？其实这些代码只是为了兼容多线程场景避免数据失效和冗余通知让逻辑更加健壮而已，对于我们普通开发者其实并不会遇到多线程中通知数据观察者的情况。只有当Lifecycle触发生命周期回调时才有可能出现多线程场景，也就是说`LifecycleBoundObserver.onStateChanged()`存在多线程的可能（原因结尾再说，不是本章重点）。

**dispatchingValue(@Nullable ObserverWrapper initiator)的2种取值**
- 页面生命周期回调中通知数据观察者时传入的是this，通过内部的代码逻辑可以看出当initiator有值时只通知给initiator这个观察者，这很好理解，当某个页面生命周期发生变化时，肯定我们只想通知在该页面中监听LiveData的数据观察者去刷新页面，没必要把这个LiveData的全部数据观察者都刷新一遍，因为其他页面中注册的数据观察者完全会在它自己所属的页面从非活跃到活跃状态切换时得到通知。
- 而当setValue(null)时是因为数据源发生了变更而发出的通知，肯定要让所有数据观察者都知道这个数据的变化，即使某些页面处于非活跃状态本次更新并不会立马被渲染到页面上那也得通知以便更新数据观察者内部的数据源以及mVersion版本号。



我们来看一下：
```
@MainThread
protected void setValue(T value) {
    必须是主线程
}

protected void postValue(T value) {
    ... 最终也是回调到setValue()所以也是必须主线程
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    必须是主线程
}

@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    必须是主线程
}
```
**照理说LiveData的所有值更新操作和设置监听都必须在主线程进行操作，那在触发通知的过程中哪里来的多线程呢？答案在于LifecycleBoundObserver这个类。**
我们知道当页面生命周期发生变化时Lifecycle会将页面的生命周期回调给LifecycleBoundObserver.onStateChanged()这个回调。通过之前[Jetpack-Lifecycle]()我们可以知道它是通过LifecycleOwner中的LifecycleRegistry来实现的页面观察者模式，正常情况下LifecycleRegistry的创建过程都是`LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);`所以其内部有个变量mEnforceMainThread = enforceMainThread;会被置为true，所以生命周期变化的通知都会在主线程中执行也就是说LifecycleBoundObserver.onStateChanged()回调也会在主线程中执行。但是LifecycleRegistry中也有个方法可以实现对象创建：
```

    /**
     * Creates a new LifecycleRegistry for the given provider, that doesn't check
     * that its methods are called on the threads other than main.
     * <p>
     * LifecycleRegistry is not synchronized: if multiple threads access this {@code
     * LifecycleRegistry}, it must be synchronized externally.
     * <p>
     * Another possible use-case for this method is JVM testing, when main thread is not present.
     */
    @VisibleForTesting
    @NonNull
    public static LifecycleRegistry createUnsafe(@NonNull LifecycleOwner owner) {
        return new LifecycleRegistry(owner, false);
    }
```
通过这个方法创建的LifecycleRegistry对象不会强制Lifecycle机制必须在主线程中执行，所以就有可能导致LifecycleBoundObserver.onStateChanged()在子线程中被回调。当然这个方法并不建议我们普通开发者使用，通常是JVM用于测试使用。



3. 






1.void activeStateChanged(boolean newActive)判断当前观察者对应的LifecycleOwner是否处于活跃状态，如果处于活跃状态则分发事件通知观察者，changeActiveCounter(mActive ? 1 : -1)只是检测LiveData的当前活跃观察者数量从0到1以及从1到0这两种情况的回调。


当页面的生命周期状态发生变化时会回调到每一个封装起来的观察者Wrapper的onStateChanged()，Wrapper内部会判断当前这个页面的生命周期是否是活跃状态，如果是则根据mVersion判断是否需要更新值，如果需要则调用onChanged。如果是DESTROY则移除该观察者，这就是Fragment为什么要使用viewLifecycleOwner的原因所在，并且当一个页面变化时，所对应的每个LiveData可能有n多个观察者，每个观察者都是这么实现。



[为什么Fragment中要使用viewLifecycleOwner代替this](https://juejin.cn/post/6915222252506054663)