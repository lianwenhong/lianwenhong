---
title: Jetpack-ViewModel
date: 2022-09-06 15:20:52
tags:
categories: Android
---
## ViewModel简介
ViewModel是Android Jetpack库中的一员，旨在以注重生命周期的方式存储和管理界面相关的数据。其与生命周期强相关。

ViewModel主要有几点需要关注的特点：
1. 在组件（Activity/Fragment）的生命周期中ViewModel的数据会一直保存在内存中，即便在组件发生重建时（例如当Activity屏幕旋转或者设置改变等原因导致的页面重建）也会一直存在。
2. ViewModel可以实现组件之间的数据共享，主要是通过使用相同的ViewModelStore来进行共享。Fragment可以通过Activity的ViewModelStore。或者⼦Fragment可以使⽤parentFragment的ViewModelStore来共享，或者也可以使⽤Activity的ViewModelStore共享；**需要注意的是Fragment中如果是使用Activity进行数据共享的话ViewModel的释放就会跟随Activity的生命周期。**

## ViewModel使用
- 正常情况下无需单独引入 ViewModel 相关库，因为androidx.appcompat:appcompat:1.4.1会自带 Lifecycle、LiveData、ViewModel 等依赖库。
- 如果想单独引入 ViewModel 或者其其它相关扩展库可如下操作：
```
// 模块的 build.gradle
// ViewModel
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.4.1'
// 用于 Compose 的 ViewModel 实用程序
implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:2.4.1'
// ViewModel 的已保存状态模块
implementation 'androidx.lifecycle:lifecycle-viewmodel-savedstate:2.4.1'

// 只有 Lifecycles（不带 ViewModel 或 LiveData）
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.4.1'
// 注释处理器
kapt 'androidx.lifecycle:lifecycle-compiler:2.4.1'
// 替换 - 如果使用 Java8，请使用此注释处理器，而不是 lifecycle-compiler 注释处理器
implementation 'androidx.lifecycle:lifecycle-common-java8:2.4.1'
// 可选 - 在 Service 中实现 LifecycleOwner 的助手
implementation 'androidx.lifecycle:lifecycle-service:2.4.1'
// 可选 - ProcessLifecycleOwner 给整个 App 前后台切换提供生命周期监听
implementation 'androidx.lifecycle:lifecycle-process:2.4.1'

// LiveData
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.4.1'
// 可选：对 LiveData 的 ReactiveStreams 支持
implementation 'androidx.lifecycle:lifecycle-reactivestreams-ktx:2.4.1'
// 可选 - LiveData 的测试助手
testImplementation 'androidx.arch.core:core-testing:2.1.0'
```
先从一个简单的使用代码来证实一下刚才说的特点1:ViewModel类让数据可在发生屏幕旋转等配置更改后继续留存

```
class MainViewModel : ViewModel() {
    var number: Int = 0
}

class MainActivity : BaseActivity<MainViewModel>() {

    private var btnPlus: Button? = null
    private var btnSub: Button? = null

    private var number = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 获取一个MainViewModel
        ViewModelProvider(this).get(MainViewModel::class.java)
        initView()
    }

    private fun initView() {
        btnPlus = findViewById(R.id.btn_plus)
        btnSub = findViewById(R.id.btn_sub)

        btnPlus?.setOnClickListener {
            number = 100
            viewModel?.number = 100
        }

        btnSub?.setOnClickListener {
            LogUtils.d("MainActivity number : $number")
            LogUtils.d("MainViewModel number : ${viewModel?.number}")
        }
    }

    /**
     * 只有manifest文件中该Activity设置android:configChanges="orientation|screenSize"时才会调用此回调，否则都是销毁重建不会走到这个回调
     */
    override fun onConfigurationChanged(newConfig: Configuration) {
        LogUtils.d(" >>> onConfigurationChanged <<<")
        super.onConfigurationChanged(newConfig)
    }
    
}
```
这个例子中，因为我们并没有将MainActivity在manifest中设置android:configChanges="orientation|screenSize"属性，所以只要切换横竖屏就会发生Activity页面重建。

上述代码意思是：在MainActivity和MainActivity对应的ViewModel中都声明了一个number属性。点击+号时会将两个number值都设置为100，然后切换横竖屏发生页面重建，此时我们看执行结果：

```
首先启动应用然后点击+号

紧接着点击-号时输出为：
2022-09-01 18:35:33.137 2918-2918/com.lianwenhong.tradition_app D/JectpackFamily:  >>> MainActivity number : 100 <<<
2022-09-01 18:35:33.137 2918-2918/com.lianwenhong.tradition_app D/JectpackFamily:  >>> MainViewModel number : 100 <<<
将应用切换为横屏，再点击-号，此时输出为：
2022-09-01 18:35:42.139 2918-2918/com.lianwenhong.tradition_app D/JectpackFamily:  >>> MainActivity number : 0 <<<
2022-09-01 18:35:42.139 2918-2918/com.lianwenhong.tradition_app D/JectpackFamily:  >>> MainViewModel number : 100 <<<
```
可见页面发生重建时写在MainActivity中的变量时重新创建的，而在ViewModel中保存的变量则未被重新创建而是被保留了下来。具体为什么我们后面将原理的时候再说。

接下来举个ViewModel共享数据的例子：

```
class MainActivity : AppCompatActivity() {
    private var txtSeekValue: TextView? = null
    private var seekBar: SeekBar? = null
    private var gotoFragment: TextView? = null
    
    private var viewModel:MainViewModel? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        viewModel = ViewModelProvider(this).get(MainViewModel::class.java)
        initView()
    }
    
    private fun initView() {

        txtSeekValue = findViewById(R.id.txt_seek_value)
        seekBar = findViewById(R.id.seek)
        gotoFragment = findViewById(R.id.btn_goto_fragment)

        demoShareData()
    }
    
    /**
     * Activity中有一个SeekBar，Fragment中也有一个SeekBar，实现两者的SeekBar值同步
     */
    private fun demoShareData() {
        seekBar?.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
                viewModel?.seekValue = progress
                txtSeekValue?.text = "${viewModel?.seekValue}"
            }

            override fun onStartTrackingTouch(seekBar: SeekBar?) {
            }

            override fun onStopTrackingTouch(seekBar: SeekBar?) {
            }
        })
        gotoFragment?.setOnClickListener {
            val mainFragment = MainFragment(true)
            if (!mainFragment.isAdded) {
                val transaction = supportFragmentManager.beginTransaction()
                transaction.add(R.id.f_me11, mainFragment)//动态添加
                transaction.addToBackStack("main_fragment")
                transaction.commit()//提交
            }
        }
    }
}

class MainViewModel : ViewModel() {

    var seekValue: Int = 0

}

class MainFragment(override val useParentViewModel: Boolean) :Fragment() {
    private var txtSeekValue: TextView? = null
    private var seekBar: SeekBar? = null
    private var fViewModel: ViewModel? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 如果传入的ViewModelStoreOwner是Activity则会从对应的Activity中获取MainViewModel名称对应的ViewModel，
        // 可以简单理解为此时fViewModel就是MainActivity中的MainViewModel
        activity?.let { fViewModel = ViewModelProvider(it)[MainViewModel::class.java] }
        // 如果如果传入的ViewModelStoreOwner是Fragment则表示用Fragment中的ViewModel，数据就和Activity中无关了，
        fViewModel = ViewModelProvider(this)[MainFragmentViewModel::class.java]
    }

    /**
     * 这里使用的ViewModel是Activity中的，所以就实现了Activity与Fragment的数据共享
     * 同样的，如果要实现Fragment和子Fragment之间的数据共享，只需要在子Fragment中使用父Fragment作为ViewModelStoreOwner
     * 代码：
     * parentFragment?.let { fViewModel = ViewModelProvider(it).get(MainViewModel::class.java) }
     * */
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val view = inflater.inflate(R.layout.fragment_main, container, false)
        txtSeekValue = view.findViewById(R.id.txt_seek_value)
        txtSeekValue?.text = "${(fViewModel as MainViewModel).seekValue}"
        seekBar = view.findViewById(R.id.seek)
        seekBar?.progress = (fViewModel as MainViewModel).seekValue
        seekBar?.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
                if (fromUser) {
                    (fViewModel as MainViewModel).seekValue = progress
                    txtSeekValue?.text = "${(fViewModel as MainViewModel).seekValue}"
                }
            }

            override fun onStartTrackingTouch(seekBar: SeekBar?) {
            }

            override fun onStopTrackingTouch(seekBar: SeekBar?) {
            }

        })
        return view
    }
}
```
## ViewModel实现原理
用几个问题来切入源码阅读
1. ViewModel是怎么创建的
2. ViewModel为什么能实现Activity重建时保留数据
3. ViewModel进行数据共享是怎么实现的

### ViewModel是怎么创建的

首先看ViewModel的创建过程，我们通常使用这种方式来获取一个ViewModel：`ViewModelProvider(this).get(MainViewModel::class.java)`，意思是从一个ViewModel提供者中get一个你想要的ViewModel类型的实例。

```
public open class ViewModelProvider // 这是ViewModelProvider的类定义

// 这是ViewModelProvider的构造方法，此时第二个参数传入的是defaultFactory(owner)，从下面方法可以看出来其实真正的factory是一个SavedStateViewModelFactory
public constructor(
    owner: ViewModelStoreOwner
) : this(owner.viewModelStore, defaultFactory(owner), defaultCreationExtras(owner))

// 调用上一个构造方法只是准备了一些参数然后调用此构造，可见此时ViewModelProvider中有一个关键变量：store、factory、defaultCreationExtras
constructor(
    private val store: ViewModelStore,
    private val factory: Factory,
    private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
) {
    ...
    public open class AndroidViewModelFactory
    private constructor(
        private val application: Application?,
        @Suppress("UNUSED_PARAMETER") unused: Int,
    ) : NewInstanceFactory() {

        public constructor() : this(null, 0)

        public constructor(application: Application) : this(application, 0)

        override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {
            return if (application != null) {
                create(modelClass, application)
            } else {
                val application = extras[APPLICATION_KEY]
                if (application != null) {
                    create(modelClass, application)
                } else {
                    if (AndroidViewModel::class.java.isAssignableFrom(modelClass)) {
                        throw IllegalArgumentException(
                            "CreationExtras must have an application by `APPLICATION_KEY`"
                        )
                    }
                    super.create(modelClass)
                }
            }
        }

        override fun <T : ViewModel> create(modelClass: Class<T>): T {
            return if (application == null) {
                throw UnsupportedOperationException(
                    "AndroidViewModelFactory constructed " +
                        "with empty constructor works only with " +
                        "create(modelClass: Class<T>, extras: CreationExtras)."
                )
            } else {
                create(modelClass, application)
            }
        }

        @Suppress("DocumentExceptions")
        private fun <T : ViewModel> create(modelClass: Class<T>, app: Application): T {
            return if (AndroidViewModel::class.java.isAssignableFrom(modelClass)) {
                try {
                    modelClass.getConstructor(Application::class.java).newInstance(app)
                } catch (e: NoSuchMethodException) {
                    throw RuntimeException("Cannot create an instance of $modelClass", e)
                } catch (e: IllegalAccessException) {
                    throw RuntimeException("Cannot create an instance of $modelClass", e)
                } catch (e: InstantiationException) {
                    throw RuntimeException("Cannot create an instance of $modelClass", e)
                } catch (e: InvocationTargetException) {
                    throw RuntimeException("Cannot create an instance of $modelClass", e)
                }
            } else super.create(modelClass)
        }

        public companion object {
            internal fun defaultFactory(owner: ViewModelStoreOwner): Factory =
                if (owner is HasDefaultViewModelProviderFactory)
                    owner.defaultViewModelProviderFactory else instance

            internal const val DEFAULT_KEY = "androidx.lifecycle.ViewModelProvider.DefaultKey"

            private var sInstance: AndroidViewModelFactory? = null
            
            @JvmStatic
            public fun getInstance(application: Application): AndroidViewModelFactory {
                if (sInstance == null) {
                    sInstance = AndroidViewModelFactory(application)
                }
                return sInstance!!
            }

            private object ApplicationKeyImpl : Key<Application>

            @JvmField
            val APPLICATION_KEY: Key<Application> = ApplicationKeyImpl
        }
    }
    ...
}
```
先对上面ViewModelProvider的构建做一个简单解释：
- 在构造ViewModelProvider时需要传入一个ViewModelStoreOwner，这是一个ViewModel容器的拥有者，后面再说。
- 紧接着传入了一个`ViewModelProvider.Factory`，这是一个ViewModel的构造工厂类，此时用的是默认工厂，而针对Activity和Fragment而言这个默认工厂的真正类型是`SavedStateViewModelFactory`，这个可以通过传入的ViewModelStoreOwner中找到答案，比如传入的是Activity的话，ComponentActivity实现了`ViewModelStoreOwner`和`HasDefaultViewModelProviderFactory`两个接口，所以通过查看ComponentActivity的`getDefaultViewModelProviderFactory()`方法可知defaultFactory其实是SavedStateViewModelFactory类型
- 最后一个参数是CreationExtras类型，和ViewModelProvider.Factory同理它也是在ComponentActivity中创建，我也没详细看这到底干嘛用的，跳过也没关系

**什么是ViewModelStoreOwner？**
```
// 这是一个ViewModel容器的持有者
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```
所以实现这个接口的类必然需要实现getViewModelStore()方法并返回一个ViewModelStore，**而ViewModelStore又是什么？**
```
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
这就是一个ViewModel容器，其内部的数据结构是个Map用于根据key值存放ViewModel并提供了几个方法用于map的管理。例如当调用`clear()`时就会清空该容器并执行容器中每个ViewModel的clear()方法，插一句：`ViewModel.clear()`用于释放自身数据，它会回调一个`onCleared()`方法用于告知开发者数据清理完成。

分析到这里我们已经有了一个ViewModel的提供者ViewModelProvider，我们需要ViewModel的时候就可以找它。那我们现在去找它，`ViewModelProvider(this).get(MainViewModel::class.java)`这个get方法被重写了：

```
@MainThread
public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
    val canonicalName = modelClass.canonicalName
        ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
    return get("$DEFAULT_KEY:$canonicalName", modelClass)
}

@MainThread
public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
    // 先尝试从ViewModelStore中获取需要的ViewModel实例
    val viewModel = store[key]
    if (modelClass.isInstance(viewModel)) {
        (factory as? OnRequeryFactory)?.onRequery(viewModel)
        // 如果实例存在直接返回，如果不存在走下面逻辑通过Factory去创建
        return viewModel as T
    } else {
        @Suppress("ControlFlowWithEmptyBody")
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    val extras = MutableCreationExtras(defaultCreationExtras)
    extras[VIEW_MODEL_KEY] = key
    // ViewModel是第一次创建的话则通过ViewModelProvider.Factory去创建一个新的ViewModel并返回，返回之前也会把它加入ViewModelStore中进行缓存，下一次如果再需要获取同类型的ViewModel时就无需再创建并且数据也能得以保留
    return try {
        factory.create(modelClass, extras)
    } catch (e: AbstractMethodError) {
        factory.create(modelClass)
    }.also { store.put(key, it) }
}
```
这部分的主要意思是当开发者需要某种类型的ViewModel时，ViewModelProvider会先去ViewModelStore这个容器中获取，如果获取不到就通过`ViewModelProvider.Factory.craete()`去创建一个ViewModel实例然后将其缓存到ViewModelStore中，此处我们解析到的ViewModelProvider.Factory是SavedStateViewModelFactory类型。而create()方法内部是通过反射生成了所需的ViewModel实例，不详细进入了。

### ViewModel为什么能实现Activity重建时保留数据
先用一张图说明一下ViewModel的生命周期
{% asset_img ViewModel生命周期.jpg ViewModel生命周期 %}

看下ComponentActivity：
```
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ContextAware,
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner,
        ActivityResultRegistryOwner,
        ActivityResultCaller,
        OnConfigurationChangedProvider,
        OnTrimMemoryProvider,
        OnNewIntentProvider,
        OnMultiWindowModeChangedProvider,
        OnPictureInPictureModeChangedProvider,
        MenuHost {
    
    public ComponentActivity() {
        ...
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                // 当Activity走到onDestroy时会走到这个回调中
                if (event == Lifecycle.Event.ON_DESTROY) {
                    // Clear out the available context
                    mContextAwareHelper.clearAvailableContext();
                    // And clear the ViewModelStore
                    // 在执行Activity的销毁过程中判断是因为配置改变导致的重建还是因为Activity被关闭导致的销毁，如果是配置改变导致的重建则不会清空ViewModel数据
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
        ...
    }
}
```
当Activity因为某些原因比如在设置页面做了某些改动或者切换横竖屏等导致Activity需要销毁重建时，isChangingConfigurations()会返回true，因此`().clear();`就不会执行，数据就能得以保留。

**那Fragment中的ViewModel是在什么时候释放的呢？**
因为代码牵扯太多就不贴了，特地画了个流程图。流程图分为init、get、destroy三个过程：
{% asset_img Fragment中使用ViewModel原理.jpg Fragment中使用ViewModel原理 %}

结论就是当Fragment执行销毁过程时执行到`Fragment.performDestroyView()`代码中处理了ViewModel的数据清理逻辑，最终具体清理的代码是：
```
final class FragmentManagerViewModel extends ViewModel {
    ...
    void clearNonConfigState(@NonNull Fragment f) {
        if (FragmentManager.isLoggingEnabled(Log.DEBUG)) {
            Log.d(TAG, "Clearing non-config state for " + f);
        }
        // Clear and remove the Fragment's child non config state
        FragmentManagerViewModel childNonConfig = mChildNonConfigs.get(f.mWho);
        if (childNonConfig != null) {
            childNonConfig.onCleared();
            mChildNonConfigs.remove(f.mWho);
        }
        // Clear and remove the Fragment's ViewModelStore
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore != null) {
            viewModelStore.clear();
            mViewModelStores.remove(f.mWho);
        }
    }
    ...
}
```
**所以在Fragment中使用ViewModel时需要注意，ViewModel在onDestroyView这个生命周期节点中已经执行了clear()相应的就会回调onCleared()，所以尽量不要在后续的onDestroy和onDetach中使用ViewModel中的数据避免出现意想不到的问题，并且尽量在onCleared()中去做一些数据释放的工作**

### ViewModel进行数据共享是怎么实现的
如果理解了上面的原理那这个问题就很好回答，以Activity共享数据给Fragment为例，其实进行数据共享的本质就是在Activity和Fragment中使用同一个ViewModel以此达到共享的效果。而我们知道每个ViewModel都会被添加到一个ViewModelStore(也就是容器)中，每个ViewModelStore又从属于一个ViewModelStoreOwner，这是背景。
{% asset_img ViewModel存储关系图.jpg ViewModel存储关系图 %}
我们在获取ViewModel都会使用这样的代码，这样对于ViewModel的创建、缓存、生命周期绑定都由系统替我们完成：
```
val viewModel = ViewModelProvider(this).get(MainViewModel::class.java)
```
不建议自己这样去创建`val viewModel = ViewModel()`，这样创建出来的ViewModel不具备数据保留、生命周期绑定等特性，也不建议自行获取ViewModelStore等方式来使用ViewModel，因为ViewModelStore的方法除了clear其他都是默认修饰符，除非反射否则你不能操作ViewModelStore，而且也一样得自己去绑定生命周期等操作。

那Fragment想要获取Activity共享的数据只需要给ViewModelProvider构造传入Activity作为ViewModelStoreOwner即可，这样Fragment得到的ViewModel是从Activity的ViewModelStore这个容器中取的ViewModel，就能达到Activity和Fragment使用的是同一份数据的效果，父Fragment和子Fragment之间共享数据也同理。

**还有一点值得注意的是，Activity的销毁和Fragment的替换都会导致当前Fragment的ViewModel执行clear()。（例如Activity A中有FragmentA，Activity销毁和Activity执行replace FragmentB都会导致FragmentA中的ViewModel执行clear() ）**