---
title: Android Context分析
date: 2022-03-28 17:55:47
categories: Android
---
## Context的使用

对于Context的使用大家并不陌生，因为在Android开发的方方面面都需要使用到Context，比如说是：

startActivity()：启动Activity、getResource():获取资源、getColor():获取色值、startService():启动服务

正是由于Context是如此的重要，所以我们在很多情况下比如自定义View，都需要传入一个Context对象才能满足编码需求，所以又牵扯出了内存泄漏等问题（比如传入了Activity当成Context，在页面生命周期已经结束时由于某些耗时等操作导致引用仍被持有导致无法及时被回收）。

既然Context是如此重要，那我们就有必要对它做个深入了解。

<!-- more -->

## Context的设计思想

Context主要有2层含义

1. 从字面上来理解他就是一个上下文对象，也就是说是一个运行的环境。可以理解成你游戏运行到一半要临时保存的进度。当你之后要接着来玩儿这个游戏的时候，可以从记录中恢复之前运行的所有元素。Context其实也有点这个意思，例如当你startActivity()跳转到某一页面时，你可以回溯到你是从哪个页面中跳转过来的。
2. 从具体的源码内容来看，其实Context只是一个抽象类，里面定义了诸多访问应用程序运行所需的接口，例如启动Activity，发送广播等等。其实Android在有意淡化进程的概念，在开发者的开发过程中，通常不需要关心我当前属于哪个进程，只需要表明意图即可，例如打电话，打开网页连接等，当调用系统服务的时候也不需要关心对应接口是属于系统哪个进程，只需要通过Context发起调用，其内部就自动帮你做好了进程之间的调度。所以Context就像是一个运行环境一样，无处不在。有了Context，你的Linux进程就摇身一变成了Android世界的公民，享有Android提供的各种服务。我们来看下Context中提供了一些什么服务：

* 获取应用资源，譬如：drawable、string、asset
* 操作四大组件，譬如：启动页面，发送广播，开启服务，打开数据库
* 操作文件目录，譬如：获取/data/分区的缓存目录getExternalCacheDir()
* 检查授予权限，譬如：checkPermission()
* 获取其他服务，譬如：包管理服务，Activity管理服务，窗口管理服务等

在应用程序中随处都可以访问这些服务，这些服务的访问入口就是Context。所以开发者不用再关系进程，而只需要关心Context提供了哪些接口即可。

> Interface to global information about an application environment. This is an abstract class whose implementation is provided by the Android system. It allows access to application-specific resources and classes, as well as up-calls for application-level operations such as launching activities, broadcasting and receiving intents, etc.

### 装饰者模式是什么

在面向对象语言(OOP)中，要为一个类拓展功能最直接的方式就是**继承**，子类可以基于父类进行扩展。但是这种方式的弊端是当要拓展的功能维度足够多，并且功能要相互叠加的时候需要拓展的子类就会越来越多。举个例子：

> 基类是衣服，需求是生产防水、透气、速干三种类型的衣服，此时需要拓展出三个子类：防水衣服、透气衣服和速干衣服。此时如果需要生产一种既防水又速干的衣服，那么得拓展出一个新类：防水速干衣服。如果需求再增加：需要生产一件保暖又速干的衣服，那么得扩展出两个新类：保暖衣服和保暖速干衣服。随着需求不断的增加，所需要拓展的子类也会随之增加。

在GOF设计模式中，把继承看成**静态**类拓展，其弊端就是随着拓展的功能的增加有可能导致子类膨胀(也就是上面例子中的问题)。所以便产生了一种**动态**类拓展的模式：**装饰者模式**。

**装饰者模式虽然在实现上和代理模式很相似，但是两者要解决的问题却是截然不同的。**

**代理模式**：1.为了隐藏代理类以保证对调用方透明做到增加安全性，2.为了实现对代理类某些功能进行用户无感知的功能增强。

**装饰者模式**：这种设计模式更多的是强调拓展性，可对任意的功能进行随机组合，有效缓解了拓展维度过多时的子类膨胀。

所以当使用装饰者模式的时候，上例中的需求我们就可以这么做：

> 基类还是衣服，需求一样是生产防水、透气、速干三种功能衣服。此时需要拓展出三个子类：防水衣服、透气衣服、速干衣服。此时要新增防水速干衣服时，只需要将防水衣服和速干衣服两者进行组合即可这样就不需要再生成新的子类，同理当需要生产保暖又速干衣服时只需要拓展一个保暖衣服即可，然后将保暖衣服和速干衣服进行组合便能满足需求。

通过上面说明我们明白了装饰者设计模式的妙处了，具体装饰者设计模式的编码实现我会再出一篇文章详细阐述。当然通过这个例子我们回归正题其实Context的实现也是使用了装饰者模式。

### Context类关系图以及装饰者模式的应用

{% asset_img Context类关系图.jpg Context类关系图 %}

从Context的类关系我们可以看出，这是一个典型的装饰者模式。

基类Context定义了各种基础功能接口，ContextImpl则负责实现接口的具体功能。

对外提供Context实现时，需要对它进行一步包装，这就有了ContextWrapper这个类。装饰类一般只是一个传递者，其内部所有的方法实现都是调用ContextImpl，所以ContextWrapper中需要持有一个ContextImpl的引用。

装饰者存在的价值就是为了拓展某个类的功能，Context已经提供了丰富的系统功能但是仍不能满足应用程序的编程需要，所以Android又拓展了一些装饰器，其中包括Application、Activity、Service。此时才能发现原来Context真的是无处不在，在Activity中调用startActivity()其实最终还是通过Context来发起的调用。那么拓展这几个装饰者的意义何在？

* Application：拓展了应用的生命周期流程控制
* Activity：拓展了单一页面的生命周期流程控制
* Service：拓展了后台服务的生命周期流程控制

它们分别对Context进行了不同维度的拓展，同时也可以将它们当成Context来使用。这就可以解释为什么你在Application中也可以启动页面，在Service中也可以启动页面。

那么既然四大组件中2大都使用的装饰者设计模式都设计成Context的装饰类，为什么**BroadcastReceiver**和**ContentProvider**不是Context的子类呢？

```
//ContentProvider的构造方法:
public ContentProvider(
        Context context,
        String readPermission,
        String writePermission,
        PathPermission[] pathPermissions) {
    mContext = context;
    mReadPermission = readPermission;
    mWritePermission = writePermission;
    mPathPermissions = pathPermissions;
}

//BroadCastReceiver的onReceiver()方法：
public abstract void onReceive(Context var1, Intent var2);

```

看代码可以看出，它们内部都需要传入一个Context，其实换个角度看它们也是装饰类，内部也包装了Context。只是因为这两大组件的使用上和Activity和Service有较大的差别，并且它们内部并不没有很复杂的生命周期控制流程等，所以它们就用了最简单的实现方式。

> **题外话：**装饰者模式存在于Android源码中很多地方，比如除了Context，Window的设计也是用的装饰者模式。

至此我们做个小结：Context在Android中无处不在，它是Android系统为了弱化进程概念而设计出来的一个上下文对象（也可以理解为代表了当前的运行环境）。Context是个抽象类其提供了各种各样的功能接口供开发者使用，例如想获取资源，想跳转页面等等，都可以通过调用Context来获取。Application、Activity、Service均是Context的装饰类，它们分别拓展了不同的功能用于针对不同的应用场景。Context的真正实现其实是在ContextImpl中。

## Context源码分析

> 源码基于API25进行讲解,代码只节选部分重要内容，具体需要自行阅读源码

Context主要有3种：
- **SystemContext：系统进程SystemServer的Context**
- **AppContext：应用进程的Context**
- **ActivityContext：Activity的Context，只有ActivityContext跟界面显示相关，需要传入activityToken和有效的DisplayId**

对于Context的源码分析我们通过2条主线来进行：

1. **Application的Context是如何构建的**
2. **Activity的Context是如何构建的**

开始前先说明一个概念：Android系统进程与应用进程之间的通信建立在Binder通信之上，而以下两个接口是Android为应用进程与系统进程之间通信而设计的：

* **IApplicationThread**: 作为系统进程请求应用进程的接口
* **IActivityManager**: 作为应用进程请求系统进程的接口

### Application的Context构建流程

{% asset_img Application的ContextImpl创建流程.jpg Application的ContextImpl创建流程 %}

整个流程分为几大块来讲解：

1. Android开启一个进程时最终是从java层的`ActivityThread.main()` 方法开始的，然后调用`ActivityThread.attach()` 方法。该方法内部会调用`ActivityManagerService.attachApplication(IApplicationThread)`，了解过Binder的就能明白**ActivityManagerService(AMS)** 其实就是Binder通信的实现，此时调用`attachApplication()` 之后就进入系统进程对应用做了一系列的初始化，然后通过传入的`ApplicationThread.bindApplication()` 将初始化的信息回调给用户进程

```
public static void main(String[] args) {
    ...
    // 主线程Lopper  
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    // 进入attach()方法中，传入false表示非系统应用
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    ...
    Looper.loop();
    ...
}
```

在attach()方法中，通过以下代码进入AMS进程对应用进行初始化

```
final IActivityManager mgr = ActivityManagerNative.getDefault();
try {
    mgr.attachApplication(mAppThread);
} catch (RemoteException ex) {
    throw ex.rethrowFromSystemServer();
}
```

此时会回调到**ActivityManagerService**类中，循着方法进入最终调用到`ActivityManagerService.attachApplicationLocked()`方法，对应用做了一系列的初始化赋值并回调给**IApplicationThread** 对象从而进入应用进程

```
thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
        profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
        app.instrumentationUiAutomationConnection, testMode,
        mBinderTransactionTrackingEnabled, enableTrackAllocation,
        isRestrictedBackupMode || !normalMode, app.persistent,
        new Configuration(mConfiguration), app.compat,
        getCommonServicesLocked(app.isolated),
        mCoreSettingsObserver.getCoreSettingsLocked());
```

2. 应用进程收到**AMS**的初始化结果之后生成一个临时的存储对象**AppBindData**并最终通过H这个Handler调用到`handleBindApplication()`方法。

```
public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

    ...

    AppBindData data = new AppBindData();
    data.processName = processName;
    data.appInfo = appInfo;
    data.providers = providers;
    data.instrumentationName = instrumentationName;
    data.instrumentationArgs = instrumentationArgs;
    data.instrumentationWatcher = instrumentationWatcher;
    data.instrumentationUiAutomationConnection = instrumentationUiConnection;
    data.debugMode = debugMode;
    data.enableBinderTracking = enableBinderTracking;
    data.trackAllocation = trackAllocation;
    data.restrictedBackupMode = isRestrictedBackupMode;
    data.persistent = persistent;
    data.config = config;
    data.compatInfo = compatInfo;
    data.initProfilerInfo = profilerInfo;
    sendMessage(H.BIND_APPLICATION, data);
}
```

3. `handleBindApplication()`中首先调用`getPackageInfoNoCheck()`创建出一个**LoadedApk**并将其缓存起来。该对象表示一个已经加载解析过的APK文件。紧接着通过PMS构造出一个**InstrumentationInfo**对象紧接着通过它使用类加载器构建出一个**Instrumentation**对象，该对象其实是对Activity或者Application方法调用的一个统一收口(简单说就是将ActivityThread对Activity和Application的通信都统一规范到这一个类中进行)，期间的通信介质就是**LoadedApk**。

```
private void handleBindApplication(AppBindData data) {
    ...
    // 构建一个LoadedApk对象，该对象其实是应用在内存中的表现形式。
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

    ...
  
    final InstrumentationInfo ii;
    if (data.instrumentationName != null) {
        try {
            // 内部是通过PackageManagerService(PMS)来创建一个InstrumentationInfo对象，用于后续生成Instrumentation
            ii = new ApplicationPackageManager(null, getPackageManager())
                    .getInstrumentationInfo(data.instrumentationName, 0);
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find instrumentation info for: " + data.instrumentationName);
        }

        mInstrumentationPackageName = ii.packageName;
        mInstrumentationAppDir = ii.sourceDir;
        mInstrumentationSplitAppDirs = ii.splitSourceDirs;
        mInstrumentationLibDir = getInstrumentationLibrary(data.appInfo, ii);
        mInstrumentedAppDir = data.info.getAppDir();
        mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
        mInstrumentedLibDir = data.info.getLibDir();
    } else {
        ii = null;
    }

    ...
    if (ii != null) {
        // 这个Context不是Application的Context，本次关注Application的context的创建所以这边咱不关心
        final ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

        try {
            final ClassLoader cl = instrContext.getClassLoader();
            // 创建出Instrumentation对象
            mInstrumentation = (Instrumentation)
                cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            throw new RuntimeException(
                "Unable to instantiate instrumentation "
                + data.instrumentationName + ": " + e.toString(), e);
        }
    
        final ComponentName component = new ComponentName(ii.packageName, ii.name);
        mInstrumentation.init(this, instrContext, appContext, component,
                data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
        ...
    } else {
        mInstrumentation = new Instrumentation();
    }

    ...
    try {
        // 调用LoadedApk的makeApplication()开始了Application的创建流程以及将其与ContextImpl绑定
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        ...
        try {
            mInstrumentation.onCreate(data.instrumentationArgs);
        }
        catch (Exception e) {
            throw new RuntimeException(
                "Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
        }

        try {
            // 执行Application.onCreate()
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```

通过上面代码可以看到我们有了**LoadedApk(代表整个APK)、Instrumentation(ActivityThread与Application和Activity的通信收口)** 这俩对象，主要创建流程都是在这俩类中进行的。
LoadedApk中：

```
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    ...

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        // 没有自定义Application或者规定了使用默认Application，则初始化的是android.app.Application
        appClass = "android.app.Application";
    }

    try {
        // 获取类加载器
        java.lang.ClassLoader cl = getClassLoader();
        ...
        // 创建出应用Context
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 执行Instrumentation的newApplication()创建一个Application
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    ...
    return app;
}
```

必要的类都创建完毕之后就该开始使用它们了，调用`LoadedApk.makeApplication()`方法来创建出一个**Application**对象(该对象之后再传递给Instrumentation来执行onCreate()等方法从而走到Application的生命周期)，创建Application的过程概括来讲就是调用类加载器将我们AndroidManifest中生命的Application加载进内存，如果我们没有指定自己的Application的话就默认会加载"**android.app.Application**"。
具体的Application创建流程是首先生成一个ClassLoader，然后通过`ContextImpl.createAppContext()`构造了一个appContext(**应用级别的Context**)，构造时保存了LoadedApk，进程的ActivityThread以及初始化了Resource资源，ApplicationContentResolver对数据库的操作类等等。

4.然后将这两个对象传入应用的`Instrumentation.newApplication()`，其内部使用类加载器+反射生成一个Application，紧接着调用`Application.attach()`将appContext设置给Application这个装饰类

```
static public Application newApplication(Class<?> clazz, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    // 反射生成Application对象
    Application app = (Application)clazz.newInstance();
    // 将应用Context设置给Application，此时其实就是设置给了ContextWrapper的mBase属性，而Application是ContextWrapper子类所以自然它也就和Context关联起来了。
    app.attach(context);
    return app;
}
```

这整个流程下来就完成了Application中Context的构建，也就是Application这个装饰类对ContextImpl的装饰。

### Activity的Context构建流程

如果从Activity的启动流程来讲解那将是一篇遥遥无期的文章，这里省去了Activity启动流程中前半部分复杂的逻辑，具体可以参考另外一个文章我会贴链接。

{% asset_img Activity的ContextImpl创建过程.jpg Activity的ContextImpl创建过程 %}

我们通过Activity启动流程中可以知道最终会调用到`ActivityThread.handleLaunchActivity()`方法中来，紧接着在其内部会执行`Activity a = performLaunchActivity(r, customIntent);` 用于创建Activity，然后调用`handleResumeActivity(...);`开始页面的测绘流程等。本篇只为探究Activity的Context创建过程，所以只关心performLaunchActivity()流程。

Activity的Context创建过程比Application中Context流程简单很多。主要分为几步(**以下代码均摘抄自performLaunchActivity()方法**)：

1. 通过ActivityThread中的mInstrumentation对象调用newActivity()生成对应的Activity对象，其内部生成原理就是通过反射创建。而这个mInstrumentation其实在应用启动过程中已经创建完毕，也就是在Application的Context创建流程中。
```
// 通过LoadedApk获取ClassLoader
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
// 创建出所需启动的Activity对象，内部是使用反射
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
```
2. 调用createBaseContextForActivity()方法去创建一个Activity对应的ContextImpl，其内部也是调用的ContextImpl的构造方法创建。
```
Context appContext = createBaseContextForActivity(r, activity);

private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
    ...
    // 创建Activity对应的ContextImpl
    ContextImpl appContext = ContextImpl.createActivityContext(
            this, r.packageInfo, r.token, displayId, r.overrideConfig);
    appContext.setOuterContext(activity);
    Context baseContext = appContext;

    ...
    return baseContext;
}
```
3. 调用activity.attach(...)方法将创建出来的ContextImpl与Activity绑定起来，当然传递的参数有很多，比如也会将Application传递进去，当然这时候这个对象也已经存在了。紧接着内部还是老配方：执行attachBaseContext(context);将ContextImpl设置给父类ContextWrapper的mBase属性。
```
// 将创建出来的ContextImpl关联给Activity，其内部是调用了attachBaseContext(context);将其设置给ContextWrapper.mBase
// 当然这个方法做了很多很多事，这里不研究别的我们只关心Context的创建流程
activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window);
```

至此流程结束

## Context注意事项

#### 内存泄漏问题：
**内存泄漏的本质是长生命周期的对象持有了短生命周期对象的引用导致短生命周期对象在无用的情况下不能及时被GC**
    
而使用Context导致内存泄漏的情况往往是将Activity这种相对短生命周期的对象传给其他对象使用，可能其他对象中有耗时操作导致Activity无法被及时回收。还有一个典型的场景就是在Android开发中往往在设计很多单例的时候需要传入一个Context，如果此时传入的是Activity那就会造成内存泄漏。因为我们知道通常单例方法是static，其涉及的生命周期是整个进程，所以为了解决这个问题可以考虑传入Application的Context来解决这个问题。
#### Context的使用：
    
在Android开发中在一个Activity中获取Context的方法有很多种：
    
- getApplication()：返回Application对象
- getApplicationContext()：与getApplication()返回同一个对象，只不过其返回的是Context类型，java中向上转型必然会被阉割掉一些子类独有的方法
- getBaseContext()：返回Activity的ContextImpl对象（Application.getBaseContext()：返回Application的ContextImpl对象）
- Activity.this：返回Activity本身

    
    正是ContextImpl被外层装饰器包装了一下才形成了Context不同功能的拓展。

## 总结

Context淡化了Android进程的概念，其提供了一个应用的运行环境。Android中它无处不在，开发者可以通过它调用一系列的系统方法比如获取资源，打开页面，打开服务等等。

其实现上采用了装饰者模式，Activity、Application、Service等都是装饰类，当开发者使用这些装饰者作为Context来使用的时候，其实真正的实现逻辑是在ContextImpl类中。

在Context的使用中要十分注意避免出现内存泄漏问题。_

---
