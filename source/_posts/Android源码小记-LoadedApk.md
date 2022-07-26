---
title: Android源码小记-LoadedApk
date: 2022-04-14 15:35:23
tags: 
categories: Android
---

文章基于API25分析，源码地址：http://androidxref.com/7.1.1_r6/

## 什么是LoadedApk
看下源码中对LoadedApk的注释：

```
/**
 * Local state maintained about a currently loaded .apk.
 * @hide
 */
public final class LoadedApk {}
```
翻译过来就是维护当前加载的 .apk 的本地状态。大白话讲LoadedApk就是一个.apk文件在内存中的表现形式。

> 换句话说LoadedApk是apk安装文件在内存中的数据，可以在LoadedApk中得到apk文件中的代码、资源文件、Activity、Service组件、manifest配置文件等等信息，这些都封装在LoadedApk这个类中。


我们知道应用启动是通过Zygote进程fork出一个应用进程之后调用ActivityThread.main()方法才走到应用自身。

**1个apk应用对应1个ActivityThread**

**1个apk应用也对应1个LoadedApk**

<!-- more -->

早前我在阅读源码的时候一直有一个疑惑，既然LoadedApk是apk文件在内存中的表现形态，那我们的应用的ActivityThread里面为什么mPackages、mResourcePackages这俩属性为什么要用map来定义呢？那不是说明会有很多个LoadedApk？而且我之前看一个文章上有一句话：*一个进程可以加载多个apk*。我一直不理解？

直到后来我了解插件化的时候我想明白了！插件化有一种方式是定义一个插件LoadedApk然后插入ActivityThread的mPackages集合中，这样在宿主工程中就能轻松访跳转插件中的Activity。原理是使用插件LoadedApk获取到插件的ClassLoader并加载出插件中的Activity类（具体原理我会再行撰文）。这就理解了一个进程是可以加载多个apk的。

既然LoadedApk是apk在内存中的表现形式，其内部包含了整个`mApplicationInfo`，那就可以理解用它能做很多事
- 创建一个Application放在这个类中再合适不过，因为LoadedApk本身就包含了manifest配置文件中的信息。
- 用它来获取Resources也合情合理，因为资源文件也包含其中。
- ...
也就是说其实在各个地方只要获取到这个LoadedApk对象就能获取应用的所有信息。


## LoadedApk创建过程源码分析

我们了解了App启动流程之后（如不了解看我另一文章 xxx）知道，一个App进程启动之后会立马调用到ActivityThread.main()方法中来，在内部会执行attach()然后调用AMS.attachApplication(IApplicationThread)方法让AMS进程去初始化应用最后将结果通过IApplication这个接口返回给应用进程。此时应用进程会通过H这个Handler走到handleBindApplication()这个方法，应用就开始处理并创建Application了，LoadedApk也是在这一步生成的。

> ActivityThread.attach() -> AMS.attachApplication(IApplicationThread thread) -> ApplicationThread.bindApplication() -> H -> ActivityThread.handleBindApplication() -> ActivityThread. getPackageInfoNoCheck(data.appInfo, data.compatInfo) -> LoadedApk.构造()


直接分析getPackageInfoNoCheck():

```
ActivityThread.java

public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,CompatibilityInfo compatInfo) {
    return getPackageInfo(ai, compatInfo, null, false, true, false);
}

private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
        ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
        boolean registerPackage) {
    // 判断应用uid，一般正常安装的apk differentUser值为false，除非是手动去加载的其他apk
    final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
    synchronized (mResourcesManager) {
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            // Caching not supported across users
            ref = null;
        } else if (includeCode) {
            // 传入的includeCode为true，所以从缓存中取出LoadedApk对象
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        // 如果是刚进应用则缓存中就不存在该apk对应的LoadedApk，就会去走创建流程
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
            if (localLOGV) Slog.v(TAG, (includeCode ? "Loading code package "
                    : "Loading resource-only package ") + aInfo.packageName
                    + " (in " + (mBoundApplication != null
                            ? mBoundApplication.processName : null)
                    + ")");
            // 创建一个新的LoadedApk对象，里面包含了ApplicationInfo，CompatibilityInfo等信息
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

            if (mSystemThread && "android".equals(aInfo.packageName)) {
                packageInfo.installSystemApplicationInfo(aInfo,
                        getSystemContext().mPackageInfo.getClassLoader());
            }

            if (differentUser) {
                // Caching not supported across users
            } else if (includeCode) {
                // 创建完成之后会将其缓存起来，后面系统流程中需要使用的时候直接根据对应的包名从缓存中获取
                mPackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            } else {
                mResourcePackages.put(aInfo.packageName,
                        new WeakReference<LoadedApk>(packageInfo));
            }
        }
        return packageInfo;
    }
}
```

这就是应用的LoadedApk的创建流程，只是简单理解不去深究里面的ApplicationInfo、CompatibilityInfo这些是如何构造的就很简单，要深究的话一篇也讲不完。

在App的启动过程中，LoadedApk做的一个很关键的事情就是创建Application对象，我们简单看看`handleBindApplication`内通过LoadedApk去创建一个Application的过程：

```
ActivityThread.java

private void handleBindApplication(AppBindData data) {
    ...
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    ...
}
```
调用到`LoadedApk.makeApplication()`方法去了：

```
public Application makeApplication(boolean forceDefaultAppClass,Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

    Application app = null;

    // 获取manifest配置文件中指定的Application，如果我们自定义了那么appClass就是我们自定义Application的类名，否则系统默认创建类名为"android.app.Application"的默认Application对象
    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        //  获取应用类加载器，正常情况下取出来的会是PathClassLoader
        java.lang.ClassLoader cl = getClassLoader();
        // 表示非系统应用
        if (!mPackageName.equals("android")) {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                    "initializeJavaContextClassLoader");
            initializeJavaContextClassLoader();
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
        // 先创建一个ContextImpl对象后续赋值给Application对应的mBase，完成Application对ContextImpl的代理，此处会将本LoadedApk实例传给Application对应的ContextImpl，其实如果了解Activity的构建的话可以去看下，Activity对应的ContextImpl中也传入了这个LoadedApk对象。正常情况如果没有加载多个apk的话，全局都是共用的这一个LoadedApk对象
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 调用ActivityThread的mInstrumentation传入类名和类加载器执行创建实例操作，内部是用的反射来实现的创建。
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + ": " + e.toString(), e);
        }
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!instrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    }

    // Rewrite the R 'constants' for all library apks.
    SparseArray<String> packageIdentifiers = getAssets(mActivityThread)
            .getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }

        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }

    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    return app;
}
```

代码注释中清楚解释了如何使用LoadedApk去构建Application。其中涉及到了Instrumentation对象，其实这个对象只是一个对Application和Activity创建和生命周期流程等操作的统一封装，这样能使代码结构更加清晰。并且其内部还有一些模拟按键的方法。

## 小结

1. LoadedApk是apk安装文件在内存中的数据，可以在LoadedApk中得到apk文件中的代码、资源文件、Activity、Service组件、manifest配置文件等等信息，这些都封装在LoadedApk这个类中。
2. 在正常情况下同一个安装包下的Activity和Application等这些组件都是公用的一个LoadedApk对象。
3. 在任何场景下只要获取了LoadedApk对象就能获取到应用的信息，例如Activity中获取资源其实也是通过ContextImpl获取了应用的LoadedApk对象实例然后去调用的`ActivityThread.getTopLevelResources();`方法。