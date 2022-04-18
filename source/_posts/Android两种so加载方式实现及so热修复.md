---
title: System.load()与System.loadLibrary()实现及so热修复
date: 2022-04-13 09:40:10
tags:
categories: Android
---
文章基于API25分析，源码地址：http://androidxref.com/7.1.1_r6/
### ClassLoader简单复习（必要前提）

在分析System类之前，先简单复习一下ClassLoader的类加载机制。

{% asset_img ClassLoader类结构关系图以及替换dexElements实现热修复原理.jpg ClassLoader类结构关系图以及替换dexElements实现热修复原理 %}

<!-- more -->

Android应用中模式使用的类加载器是**PathClassLoader**。想要了解可简单看下**Application.getClassLoader()** 的调用栈。


```
Application.getClassLoader() -> ContextImpl.getClassLoader ->
LoadedApk.getClassLoader() -> ... -> ClassLoader.createSystemClassLoader() -> new PathClassLoader(...)
```
而**PathClassLoader**只是重载了几个构造方法，真正的代码实现都在父类**BaseDexClassLoader**中。

BaseDexClassLoader中pathList:DexPathList类说明：

```
final class DexPathList {
    ...
    
    // List of dex/resource (class path) elements.意思是class文件转化结果:dex文件集合
    private Element[] dexElements;

    // List of native library path elements. 意思是native层so文件集合  
    // 在热修复技术中so修复可以将新的so库路径追加到此数组之前达到热修效果，其实它是将例如xxx/xxx/abc.so分别以dir:xxx/xxx/、zip:abc.so这样区分存储而已，为了使用方便
    private final Element[] nativeLibraryPathElements;

    // List of application native library directories. 意思是native层so文件的路径集合，只表示路径。
    private final List<File> nativeLibraryDirectories;

    // List of system native library directories.意思是系统so文件路径集合
    private final List<File> systemNativeLibraryDirectories;

    ...
}

static class Element {
    private final File dir; // 路径
    private final boolean isDirectory; // 该Element是否是文件夹
    private final File zip; // 路径下对应的文件
    private final DexFile dexFile; // 路径下对应的dex文件
}
```

类加载机制了解简单了解完开始分析**System**类。

### System.load():

```
public static void load(String filename) {
    Runtime.getRuntime().load0(VMStack.getStackClass1(), filename);
}
```
调用Runtime类中的load0()方法将文件名传入：

```
synchronized void load0(Class fromClass, String filename) {
    if (!(new File(filename).isAbsolute())) {
        throw new UnsatisfiedLinkError(
            "Expecting an absolute path of the library: " + filename);
    }
    if (filename == null) {
        throw new NullPointerException("filename == null");
    }
    // 最终调用此方法去load so文件，其内部是去调用native方法
    String error = doLoad(filename, fromClass.getClassLoader());
    if (error != null) {
        throw new UnsatisfiedLinkError(error);
    }
}

/**
 * 注释大概意思是我们日常应用进程是从Zygote进程fork出来的所以都是公用的同一个LD_LIBRARY_PATH路径。
 * 开放这个方法是为了用户能动态的加载不同路径下的so库而不是只有启动的时候从单一的目录加载一次。
 * 理由是因为一个进程可能会运行多个apk并且用户可能会手动实现自己的BaseDexClassLoader，
 * 所以有必要为用户提供一个差异化so加载的能力
 */
private String doLoad(String name, ClassLoader loader) {
    // Android apps are forked from the zygote, so they can't have a custom LD_LIBRARY_PATH,
    // which means that by default an app's shared library directory isn't on LD_LIBRARY_PATH.

    // The PathClassLoader set up by frameworks/base knows the appropriate path, so we can load
    // libraries with no dependencies just fine, but an app that has multiple libraries that
    // depend on each other needed to load them in most-dependent-first order.

    // We added API to Android's dynamic linker so we can update the library path used for
    // the currently-running process. We pull the desired path out of the ClassLoader here
    // and pass it to nativeLoad so that it can call the private dynamic linker API.

    // We didn't just change frameworks/base to update the LD_LIBRARY_PATH once at the
    // beginning because multiple apks can run in the same process and third party code can
    // use its own BaseDexClassLoader.

    // We didn't just add a dlopen_with_custom_LD_LIBRARY_PATH call because we wanted any
    // dlopen(3) calls made from a .so's JNI_OnLoad to work too.

    // So, find out what the native library search path is for the ClassLoader in question...
    String librarySearchPath = null;
    if (loader != null && loader instanceof BaseDexClassLoader) {
        BaseDexClassLoader dexClassLoader = (BaseDexClassLoader) loader;
        // 调用BaseDexClassLoader中的方法用于将所有native的lib路径以：拼接在一起然后给native层去load so文件
        librarySearchPath = dexClassLoader.getLdLibraryPath();
    }
    // nativeLoad should be synchronized so there's only one LD_LIBRARY_PATH in use regardless
    // of how many ClassLoaders are in the system, but dalvik doesn't support synchronized
    // internal natives.
    synchronized (this) {
        // 最终调用native方法去load so库。Android所有的so库都是通过这个方法加载进内存的
        return nativeLoad(name, loader, librarySearchPath);
    }
}

// TODO: should be synchronized, but dalvik doesn't support synchronized internal natives.
    private static native String nativeLoad(String filename, ClassLoader loader,
                                            String librarySearchPath);

```
以上是**System.load()** 的全部流程。

### System.loadLibrary()

System.loadLibrary()相比System.load()会复杂一些。因为它不再是从你随便指定的路径中去加载so，而是从系统给你的路径中加载。

```
public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
}

```
一样是走到Runtime类中

```
synchronized void loadLibrary0(ClassLoader loader, String libname) {
    if (libname.indexOf((int)File.separatorChar) != -1) {
        // 如果传入的是带/的文件路径则抛异常，这个类型的异常如果进行ndk相关开发肯定见过
        throw new UnsatisfiedLinkError(
"Directory separator should not appear in library name: " + libname);
    }
    String libraryName = libname;
    if (loader != null) {

        // 从类加载器中查找so文件名并以全路径返回，类加载器中查找so时的路径是在其构造的时候指定的。
        String filename = loader.findLibrary(libraryName);
        if (filename == null) {
            // It's not necessarily true that the ClassLoader used
            // System.mapLibraryName, but the default setup does, and it's
            // misleading to say we didn't find "libMyLibrary.so" when we
            // actually searched for "liblibMyLibrary.so.so".
            throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                           System.mapLibraryName(libraryName) + "\"");
        }
        // 找到就加载
        String error = doLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }

    // 如果类加载器为空，则获取到比如xxx.so，然后从系统提供的目录中查找(例如：/vendor/lib:/system/lib)，一样判断是否只读如果是则调用doLoad()加载
    String filename = System.mapLibraryName(libraryName);
    List<String> candidates = new ArrayList<String>();
    String lastError = null;
    for (String directory : getLibPaths()) {
        String candidate = directory + filename;
        candidates.add(candidate);

        if (IoUtils.canOpenReadOnly(candidate)) {
            String error = doLoad(candidate, loader);
            if (error == null) {
                return; // We successfully loaded the library. Job done.
            }
            lastError = error;
        }
    }

    if (lastError != null) {
        throw new UnsatisfiedLinkError(lastError);
    }
    throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
}
```

当类加载器不是空的时候，调用` load.findLibrary(libraryName)`最终会调用到DexPathList中去:
```
public String findLibrary(String libraryName) {
    String fileName = System.mapLibraryName(libraryName);

    for (Element element : nativeLibraryPathElements) {
        String path = element.findNativeLibrary(fileName);

        if (path != null) {
            return path;
        }
    }

    return null;
}
```
所以这里我们就可以跟做Dex补丁一样的方式在这个Element数组前面插入补丁SO文件, 这样在findLibrary的时候就会优先返回插入的SO文件, 并执行doLoad加载插入的SO文件. 那插入的时机是什么时候? findLibrary的动作是在调用了System.loadLibrary后才执行的,所以插入补丁的动作应该是要放在System.loadLibrary之前才能确保加载的时候更新SO文件.

所以到最后其实System.loadLibrary()只是从特定的路径中去加载so库最终还是回调给Runtime.doLoad()，然后再调用Runtime.nativeLoad()这个native方法来执行的so加载。


### 小结
System.load()和System.loadLibrary()方法其实最终原理都一样，是通过调用Runtime.doLoad()最终Runtime.nativeLoad()调用到native层实现的so库加载，两者的区别在于查找so库的路径上。

关于热更新so库有两种实现
1. 和dex热更新一样，将补丁包的路径插入到DexPathList.nativeLibraryPathElements属性的最前面达到抢先加载的目的。这样的好处是开发者在加载so库的时候和往常一样直接System.loadLibrary(xxx)即可，缺点是需要考虑到Android系统不断在更新可能导致的api变动，毕竟这种方案是通过反射来修改的值。并且还需要考虑不同cpu架构插入不同so的问题，比如arm/intel
2. 使用System.load(xxx)去直接加载补丁包路径下的so，这种方式的好处在于你不需要去关心系统版本兼容和cpu架构等问题，缺点是开发者使用的时候需要改成System.load(xxx)这种加载方式，对于开发者不够透明。

附上so热修复实现图：

{% asset_img Tinker热修复so修复原理图.png Tinker热修复so修复原理图 %}


参考文章：https://blog.csdn.net/l2show/article/details/53573945