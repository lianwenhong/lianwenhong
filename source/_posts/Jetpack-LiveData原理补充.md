---
title: Jetpack-LiveData原理补充
date: 2022-09-16 18:19:34
tags:
categories:
---
本篇主要分析LiveData的三个问题，阅读本文需要实现对LiveData的原理有一定的了解，如果不了解可以参考上一篇文章：[Jetpack-LiveData](https://lianwenhong.top/2022/09/15/Jetpack-LiveData/)

### LiveData粘性事件
可以尝试一下在LiveData数据更新之后再去注册数据观察者监听，你会发现数据观察者依然会收到最新值的通知：
```
btnChangeInMain?.setOnClickListener {
    viewModel?.userName?.value = "MAIN"
    viewModel?.userName?.observe(this) { txtName?.text = "NAME:$it" }
    // viewModel?.userName?.observeForever{txtName?.text = "NAME:$it"}
}
```
那是因为LiveData处理了这种粘性事件
- 在observe()过程中因为`owner.getLifecycle().addObserver(wrapper);`会给observer和LifecycleOwner做页面生命周期绑定，而Lifecycle本身不管你在哪个生命周期中执行addObserver(wrapper)都会将所有生命周期状态回调通知给生命周期观察者（具体代码在LifecycleRegistry->sync()），也就是说Lifecycle本身就具备粘性观察的能力，所以不管时机注册数据观察者都会在其内部的onStateChanged()方法收到最新的生命周期回调并执行通知数据观察者的逻辑。
- observeForever()是因为其内部直接执行了`wrapper.activeStateChanged(true);`将最新的值通知给了数据观察者


### Fragment中使用LiveData为什么要传入viewLifecycleOwner?
在Fragment中对LiveData对象调用observe方法时，如果传递的LifecycleOwner参数为this即Fragment时，会收到Android Studio的红色波浪线提示建议我们使用viewLifecycleOwner。其实我们忽略这个报错直接运行也是没问题的因为Fragment和viewLifecycleOwner的类型FragmentViewLifecycleOwner都实现了LifecycleOwner接口。但是这个红色波浪线提示是Google有意为之，原因是因为viewLifecycleOwner的生命周期和Fragment的生命周期并不相同。

```
void performCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
        @Nullable Bundle savedInstanceState) {
    ...
    mPerformedCreateView = true;
    mViewLifecycleOwner = new FragmentViewLifecycleOwner(this, getViewModelStore());
    mView = onCreateView(inflater, container, savedInstanceState);
    ...
}

void performDestroyView() {
    mChildFragmentManager.dispatchDestroyView();
    if (mView != null && mViewLifecycleOwner.getLifecycle().getCurrentState()
                    .isAtLeast(Lifecycle.State.CREATED)) {
        // 此处回调到LiveData->LifecycleBoundObserver->onStateChanged()时状态为DESTROYED所以LiveData会将绑定了mViewLifecycleOwner的数据观察者移除
        mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    }
    mState = CREATED;
    mCalled = false;
    onDestroyView();
    ...
}

void performDestroy() {
    mChildFragmentManager.dispatchDestroy();
    // Fragment在此时才会给生命周期观察者发送ON_DESTROY事件
    // 如果此时LiveData使用Fragment执行observe()方法那么也就是到这里才会执行上述的移除数据观察者操作
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    mState = ATTACHED;
    mCalled = false;
    mIsCreated = false;
    onDestroy();
    ...
}
```
可以看到，mViewLifecycleOwner的创建时机在Fragment的onCreateView()，并且在Fragment的onDestroyView()生命周期之前就执行了`mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);`通知生命周期观察者页面走到ON_DESTROY状态，对于LiveData来说就是接收到页面处于从活跃到非活跃状态变化的通知此时就会将该页面中绑定的数据观察者执行removeObserver()操作。而Fragment则是在performDestroy()方法中也就是onDestroyView()之后onDestroy()之前才执行`mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY)`。

在Fragment没有执行addToBackStack()操作的时候其实LiveData.observe(viewLifecycleOwner,observer)和LiveData.observe(this,observer)没有什么区别，只有在Fragment执行了addToBackStack()问题才会暴露出来，先了解一下Fragment的生命周期：
{% asset_img Fragment生命周期.png Fragment生命周期 %}
**Fragment入back stack的过程会执行onDestroyView但不执行之后的onDestroy与onDetach，而出back stack是从onCreateView开始执行，而没有之前的onAttach与onCreate。**

所以结合Fragment的生命周期就能看出区别了。

假如我们在Fragment1的onCreateView()到onDestroyView()之间的任意生命周期执行如下代码：
```
// Fragment1.kt 

liveData.observe(this) {
    LogUtils.d("数据变化监听:$it")
}
```
当Fragment1被replace为Fragment2时执行了addToBackStack()将Fragment1加入回退栈back stack。此时执行的是Fragment1的入栈操作所以并不会走到onDestroy()生命周期所以LiveData收不到页面的DESTROY生命周期监听也就不会去执行removeObserver()操作。当Fragment2点击返回回到Fragment1时Fragment1又执行了onCreateView()生命周期，这样在onCreateView()到onDestroyView()之间执行的LiveData数据观察者绑定又被执行了一次，此时又由于每次observe时绑定的Observer对象都不是同一个就会导致同一个LiveData其实是被重复绑定了数据观察者。

### 为什么LiveData->setValue()->dispatchingValue()过程中会有多线程的场景兼容？
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

---
参考文章：[为什么Fragment中要使用viewLifecycleOwner代替this](https://juejin.cn/post/6915222252506054663)