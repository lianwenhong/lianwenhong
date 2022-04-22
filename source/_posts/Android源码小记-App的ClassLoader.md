---
title: Android源码解析-App的ClassLoader
date: 2022-04-14 15:36:02
tags:
categories: Android
---

## Android中的默认类加载器：

Android虚拟机就是一个特殊的JVM，不管是以前的dalvik还是现在的ART。所以其类加载流程一样遵循jvm的规则（双亲委托机制）

如果对jvm的类加载流程不熟悉可以阅读另一篇文章：[java类加载原理(ClassLoader工作原理)](http://lianwenhong.top/2022/04/16/java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%8E%9F%E7%90%86-ClassLoader%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/#more)

Android类加载器之相较于Java类加载器的最大区别在于：

java类加载器是从某个目录中去直接加载.class文件或者是从目录下的zip、jar包等归档文件中加载.class文件

而Android类加载器是从某个目录下去直接加载.dex文件或者是从目录下的zip、jar、apk等文件中的.dex文件中去加载.class文件，如果不是.dex文件的话加载不了。

<!-- more -->

{% asset_img Android默认类加载器关系图.jpg Android默认类加载器关系图 %}

以上类关系图清楚的描述了Android中的默认类加载器以及它们的父子关系。
图中有涉及到了DexPathList这个类简单来说是存放该类加载器的加载区域中的所有dex文件，在Android的插件化方案和热修复方案中经常会使用将补丁包或者插件包中的dex插入这个数组的第一个来实现热修复或者加载插件的功能。具体详情可看另一个文章：[android热修复小记](http://lianwenhong.top/2022/04/13/android%E7%83%AD%E4%BF%AE%E5%A4%8D%E5%B0%8F%E8%AE%B0/#more)

## app中的类加载器

在Android开发中例如在Activity中会调用getClassLoader()来获取类加载器，这时候这个类加载器是怎么来的呢？来分析一下

Activity类中并没有getClassLoader()，从继承关系上可以知道最终调用的是`ContextWrapper.getClassLoader()`，最终的实现在**ContextImpl**中。如果不知道的话可以看下我另外一篇文章：[Android源码小记-Context解析](http://lianwenhong.top/2022/03/29/Android%E6%BA%90%E7%A0%81%E5%B0%8F%E8%AE%B0-Context%E8%A7%A3%E6%9E%90/#more)，可以清楚了解Context的继承关系以及Activity和Context之间是如何关联以及与ContextImpl之间的联系。

```
ContextImpl.java

public ClassLoader getClassLoader() {
    return mPackageInfo != null ?
            mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader();
}
```
这个mPackageInfo对象是一个LoadedApk，持有apk的所有信息，是apk在内存中的表现形态。（[ANdroid源码小记-LoadedApk](http://lianwenhong.top/2022/04/14/Android%E6%BA%90%E7%A0%81%E5%B0%8F%E8%AE%B0-LoadedApk/)）,而它是在构造ContextImpl的时候传进来并赋值的。

在启动Activity时，最终会走ActivityThread中的handleLaunchActivity()方法，这个方法里就会最终调用到先知性Instrumentation.newActivity()通过类加载器创建一个Activity实例，然后调用createBaseContextForActivity()去创建一个处理提供Activity运行环境的ContextImpl，此时就会给mPackageInfo赋值。

```
private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
    int displayId = Display.DEFAULT_DISPLAY;
    try {
        displayId = ActivityManagerNative.getDefault().getActivityDisplayId(r.token);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }

    // 创建提供Activity运行环境的ContextImpl用于之后attach给Activity，此时传入的LoadedApk是r.packageInfo
    ContextImpl appContext = ContextImpl.createActivityContext(
            this, r.packageInfo, r.token, displayId, r.overrideConfig);
    appContext.setOuterContext(activity);
    Context baseContext = appContext;

    final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
    // For debugging purposes, if the activity's package name contains the value of
    // the "debug.use-second-display" system property as a substring, then show
    // its content on a secondary display if there is one.
    String pkgName = SystemProperties.get("debug.second-display.pkg");
    if (pkgName != null && !pkgName.isEmpty()
            && r.packageInfo.mPackageName.contains(pkgName)) {
        for (int id : dm.getDisplayIds()) {
            if (id != Display.DEFAULT_DISPLAY) {
                Display display =
                        dm.getCompatibleDisplay(id, appContext.getDisplayAdjustments(id));
                baseContext = appContext.createDisplayContext(display);
                break;
            }
        }
    }
    return baseContext;
}
```
这个**r.packageInfo**其实就是在应用刚启动时在**ActivityThread.handleBindApplication()**方法中的**data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);** 这里创建的，这时候创建出一个表示本应用的LoadedApk,正常情况下全局都共用这个LoadedApk对象，创建Activity的ContextImpl时也不例外。

回到ContextImpl.getClassLoader()中，mPackageInfo已经知道怎么来了，那再来看代码：

```
public ClassLoader getClassLoader() {
    return mPackageInfo != null ?
            mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader();
}
```
此时mPackageInfo不为空走`mPackageInfo.getClassLoader()`.

```
public ClassLoader getClassLoader() {
    synchronized (this) {
        if (mClassLoader == null) {
            createOrUpdateClassLoaderLocked(null /*addedPaths*/);
        }
        return mClassLoader;
    }
}
```
正常走到这里因为我们的应用报名肯定不是"android"，因为packageName.equals("android")是系统本身，所以我们最终只能走到`createOrUpdateClassLoaderLocked()`方法：

```
private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {
    if (mPackageName.equals("android")) {
        // 系统的情况不看，我们走不到
        ...

        return;
    }

    ...
    if (!mIncludeCode) {
        if (mClassLoader == null) {
            StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskReads();
            // 创建ClassLoader
            mClassLoader = ApplicationLoaders.getDefault().getClassLoader(
                "" /* codePath */, mApplicationInfo.targetSdkVersion, isBundledApp,
                librarySearchPath, libraryPermittedPath, mBaseClassLoader);
            StrictMode.setThreadPolicy(oldPolicy);
        }

        return;
    }

    ...
    if (mClassLoader == null) {
        ...

        mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip,
                mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
                libraryPermittedPath, mBaseClassLoader);

        ...
    }
    
    if (addedPaths != null && addedPaths.size() > 0) {
        final String add = TextUtils.join(File.pathSeparator, addedPaths);
        // 添加主工程的源码路径
        ApplicationLoaders.getDefault().addPath(mClassLoader, add);
        // Setup the new code paths for profiling.
        needToSetupJitProfiles = true;
    }

    ...
}
```
最后的最后还是通过`ApplicationLoaders.getClassLoader()`来创建：

```
public ClassLoader getClassLoader(String zip, int targetSdkVersion, boolean isBundled,
                                  String librarySearchPath, String libraryPermittedPath,
                                  ClassLoader parent) {
    // 传入的是系统ClassLoader，阅读源码其实是PathClassLoader，可以看ClassLoader.createSystemClassLoader()，其parent是BootClassLoader，所有此时baseParent = BootClassLoader
    ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();

    synchronized (mLoaders) {
        if (parent == null) {
            parent = baseParent;
        }

        
        // 如果是lib库中的类返回的加载区域为lib目录的类加载器
        if (parent == baseParent) {
            ClassLoader loader = mLoaders.get(zip);
            if (loader != null) {
                return loader;
            }

            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);

            PathClassLoader pathClassloader = PathClassLoaderFactory.createClassLoader(
                                                  zip,
                                                  librarySearchPath,
                                                  libraryPermittedPath,
                                                  parent,
                                                  targetSdkVersion,
                                                  isBundled);

            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "setupVulkanLayerPath");
            setupVulkanLayerPath(pathClassloader, librarySearchPath);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            mLoaders.put(zip, pathClassloader);
            return pathClassloader;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
        // 应用主工程中的类返回的是加载区域为apk的sourceDir所在目录的PathClassLoader
        PathClassLoader pathClassloader = new PathClassLoader(zip, parent);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        return pathClassloader;
    }
}
```
最后可以得出结论：首先反正返回的肯定是PathClassLoader，调用getClassLoader()的类属于不同的位置（主工程或者lib工程)则创建的PathClassLoader对应的加载区域也是不同的，具体要看住工程的加载区域可以跟踪一下，在创建的时候传入的zip是""，则它是通过addPath()方法传入的，该方法在BaseDexClassLoader中，而创建出来的PathClassLoader调用这个方法的地方是在ApplicationLoader.addPath():

```
void addPath(ClassLoader classLoader, String dexPath) {
    if (!(classLoader instanceof PathClassLoader)) {
        throw new IllegalStateException("class loader is not a PathClassLoader");
    }
    final PathClassLoader baseDexClassLoader = (PathClassLoader) classLoader;
    baseDexClassLoader.addDexPath(dexPath);
}
```
该方法在刚才的`LoadedApk.createOrUpdateClassLoaderLocked()`中：

```
if (addedPaths != null && addedPaths.size() > 0) {
    final String add = TextUtils.join(File.pathSeparator, addedPaths);
    ApplicationLoaders.getDefault().addPath(mClassLoader, add);
    // Setup the new code paths for profiling.
    needToSetupJitProfiles = true;
}
```
继续追溯add参数的来源是LoadedApk.makePaths(),最后的结论就是这个路径是ApplicationInfo的sourceDir。

## 小结：
在Activity中调用getClassLoader()返回的是PathClassLoader,它的父类加载器是系统的类加载器也是个PathClassLoader，再往上是BootClassLoader。

温馨提示：

要清楚明白的知道ActivityThread、Activity、Application、Context、LoadedApk、Instrumentation这些类的关系对阅读Android源码很有帮助。