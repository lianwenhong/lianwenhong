---
title: Android Context分析
date: 2022-03-28 17:55:47
categories: Android
---
### Context的使用

对于Context的使用大家并不陌生，因为在Android开发的方方面面都需要使用到Context，比如说是：

startActivity()：启动Activity、getResource():获取资源、getColor():获取色值、startService():启动服务

正是由于Context是如此的重要，所以我们在很多情况下比如自定义View，都需要传入一个Context对象才能满足编码需求，所以又牵扯出了内存泄漏等问题（比如传入了Activity当成Context，在页面生命周期已经结束时由于某些耗时等操作导致引用仍被持有导致无法及时被回收）。

既然Context是如此重要，那我们就有必要对它做个深入了解。

<!-- more -->

### Context的设计思想

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

### Context源码分析

对于Context的源码分析我们通过2条主线来进行：

1. **Application的Context是如何构建的**
2. **Activity的Context是如何构建的**

### Context注意事项

### 总结

---
