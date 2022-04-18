---
title: android热修复小记
date: 2022-04-12 17:55:47
categories: Android
---
## 认识热修复

由于Android应用程序需要手动安装apk包的特性，如果应用发布上线之后发现有bug只能再下载新包来修复。这样一来影响用户体验，二来是否下载新包的主动权在用户，所以开发者对bug无法把控。

## 主流热修复方案

1. 腾讯Tinker
2. 腾讯QZone
3. 阿里sophix
4. 美团Robust

<!-- more -->


做一些简单对比


| 方案对比        | Tinker   | QZone | Sophix       | Robust |
| ----------------- | ---------- | ------- | -------------- | -------- |
| 即时生效        | no       | no    | yes          | yes    |
| 类替换(dex插队) | yes      | no    | yes          | no     |
| So修复          | yes      | no    | yes          | no     |
| 资源修复        | yes      | yes   | yes          | no     |
| 版本支持        | all      | all   | all          | all    |
| 是否开源        | yes      | no    | no           | yes    |
| 收费情况        | 基础免费 | 免费  | 到达阈值收费 | 免费   |

### Tinker

腾讯自研的需要**重启生效**的java层实现的热修复方案

类修复是通过将原有的dex文件和修复之后的patch.dex文件进行dexdiff算法生成一个差分包下发给端，端收到差分包之后通过DexPatch算法再将差分包与端上的bug dex进行合并形成新的dex并替换掉dexElements数组来实现的。

资源修复过程构造一个新的AssetManager，并通过反射调用addAssetPath，把这个完 整的新资源包加入到AssetManager中。这样就得到了一个含有所有新资源的AssetManager。然后找到所有之前引用到原有AssetManager的地方通过反射将它们替换为这个新的AssetManager。

so的修复过程最简单，目前支持2种方式：

> 1. Hack方式：通过反射将补丁so插入到类加载器(这里我们拿到的就是BaseDexClassLoader)的DexPathList.nativeLibraryPathElements的数组最前面,这样做的好处是开发者只需像往常一样使用System.loadLibrary()来加载so，对开发者完全透明。缺点是是线上需要考虑系统版本兼容，毕竟用到了反射。
> 2. System.load()方式：可以加载任意路径下的so，好处是没有用到反射所以没有系统版本兼容问题，但是开发者使用时候加载so库的时候就需要调用System.load()方式而不是System.loadLibrary()。

> 关于so的加载load逻辑可以参考文章：http://lianwenhong.top/2022/04/13/Android两种so加载方式实现及so热修复/#more

CustomTinker代码仓库：https://github.com/lianwenhong/CustomTinker.git
本工程模拟了dexElements插队实现类修复、AssetManager替换实现资源修复、未模拟so库修复，并且未做差分包，补丁校验等逻辑。仅用于学习。

### QZone

Multidex方案是基于ClassLoader的**纯java实现**的**重启生效**的热修复方案。

其实现原理与Tinker基本相同，不同点在于Tinker的补丁包是差分包而Multidex是全量包。并且它不支持so库修复。

### Robust

Robust是**即时生效**的**Java层实现**的热修复实现方案.

其参考了google的InstantRun方案。https://www.jb51.net/article/216170.htm#_label0

其主要设计思想就是字节码动态插桩(javassist)，在编译打包时都会通过javassist在程序每个方法里都插入这样一段代码：

```
if (changeQuickRedirect != null) {
    //
    return 修复的实现
}
```

javassist原理：在class转dex的过程中会调用Transform，在该时机修改class对象，完成代码的注入。

当changeQuickRedirect不为空的时候，该方法就会命中`if(changeQuickRedirect != null)`，从而执行修复的实现代码。当为空的时候，则正常执行原逻辑。

在发生bug之后修复完bug方法时需要在方法上打上@Modify注解，这样在补丁包生成的时候该方法会被加入补丁描述类中。

为什么Robust能即时生效？

在补丁下载完成之后使用DexClassLoader将其加载进内存，然后反射将bug类对应的changeQuickRedirect复制为补丁对象。这样当程序执行到bug方法的时候就会命中changeQuickRedirect!=null然后去走补丁中的修复之后的方法。定位bug类和bug方法就是通过之前打注解时自动将其加入补丁描述类中来实现的。

当前Robust支持的动态插桩技术有AspectJ、javassist两种。

javassist demo：https://github.com/lianwenhong/JavassistDemo.git

AspectJ demo：https://github.com/lianwenhong/AspectJDemo.git

### sophix

在native动态替换java层的方法，通过native层hook java层的代码。

没研究所以不做深入探究

思考：

四大组件的热修复？

资源热修复时会不会有冲突？（系统资源package id 0x01，应用资源package id 0x7f）

参考文章：https://www.jb51.net/article/216170.htm#_label0
