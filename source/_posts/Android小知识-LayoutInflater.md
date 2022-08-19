---
title: Android小知识- LayoutInflater
date: 2022-08-19 10:49:17
tags:
categories: Android
---
### 报错引发的LayoutInflater使用上的思考
今天在开发过程中无意间收获了这么一个崩溃：

```
java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
        at android.view.ViewGroup.addViewInner(ViewGroup.java:4937)
        at android.view.ViewGroup.addView(ViewGroup.java:4768)
        at android.view.ViewGroup.addView(ViewGroup.java:4708)
        at androidx.fragment.app.FragmentStateManager.addViewToContainer(FragmentStateManager.java:840)
        at androidx.fragment.app.FragmentStateManager.createView(FragmentStateManager.java:529)
        at androidx.fragment.app.FragmentStateManager.moveToExpectedState(FragmentStateManager.java:261)
        at androidx.fragment.app.FragmentManager.executeOpsTogether(FragmentManager.java:1890)
        at androidx.fragment.app.FragmentManager.removeRedundantOperationsAndExecute(FragmentManager.java:1808)
        at androidx.fragment.app.FragmentManager.execPendingActions(FragmentManager.java:1751)
        at androidx.fragment.app.FragmentManager$5.run(FragmentManager.java:538)
        at android.os.Handler.handleCallback(Handler.java:790)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6494)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)
```
崩溃的意思是某个View已经存在父View，如果要重复添加的话就得先执行`removeView()把`它从父View中移除，这是一个很常见的错误，一个View只能存在一个父View。同理：同一个鸡蛋不可能同时被放在两个篮子中。

报错代码在一个`Fragment->onCreateView()`声明周期回调中：

```
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    return inflater.inflate(R.layout.fragment_me, container)
}
```
只需要将` inflater.inflate(R.layout.fragment_me, container)`这行改成` inflater.inflate(R.layout.fragment_me, container, false)`即可正常运行。

### LayoutInflater原理解析

**这个inflate方法旨在获取某个指定的View**

**进去翻了翻这段代码的源码，简单记录一下，如果不感兴趣可直接翻到末尾总结中查看使用时的注意事项：**

首先不管它如何创建，直接从报错代码切入，inflate方法总共有4个重载：

```
// 方法1
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}

// 方法2
public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
    return inflate(parser, root, root != null);
}

// 方法3
// attachToRoot表示是否将resource解析出来的View添加到root容器中
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    ...
    return inflate(parser, root, attachToRoot);
    ...
}

// 方法4
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    ...
}
```
在日常业务开发中比较常用的是方法1和方法3这两个
```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    // 如果root容器是null，那肯定不能将获得的view添加到一个空容器中
    // 如果root容器不是空，那直接将获取到的view添加到该容器中
    return inflate(resource, root, root != null);
}

// 方法3
// attachToRoot表示是否将resource解析出来的View添加到root容器中
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    // 这个方法是Android10新增的一个编译优化，可将xml预编译成dex，然后通过反射生成对应的View，从而减少XmlPullParser解析Xml的时间。无需关心，因为目前开关都关着，即使开起来目前来看也是一段有bug的代码！！！
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        // 如果开启了预编译并且解析成功了，直接返回对应的view
        return view;
    }
    // 通过Resources获取xml解析器，可见方法2区别其实就是在外部获取完这个xml解析器传进来而已
    XmlResourceParser parser = res.getLayout(resource);
    try {
        // 执行解析并生成view
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```
`public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) `表示解析并生成resource这个id对应的view，如果传入的父容器不是空的就顺带将这个view添加到父容器中。

`public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)`这就让开发者可以更灵活的控制view是否被添加到父容器，当然前提是root不是空的才能被添加进去。毕竟鸡蛋放在一个没有底的篮子里最终的结果就是蛋碎。

所以最终的重担还是落到方法4中。


```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        ...
        // 把root赋值给result，这里涉及到后续应该给调用方返回什么结果，后面说
        View result = root;

        try {
            advanceToRootNode(parser);
            // 取到制定resource id对应的最根view的名称
            final String name = parser.getName();
            
            // merge标签处理逻辑
            if (TAG_MERGE.equals(name)) {
                // merge标签的意义在于减少层级，而减少层级的实现是将merge标签下的子view直接加到父容器中以此减少一个层级，否则的话每个xml是不是还得写个容器来承载xml中的所有view，所以如果此时指定的xml根布局是个merge标签那就必然需要指定root并且指定其将其加入root容器，否则它将无所依附，所以此处抛出异常警告开发者
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                // 如果是merge标签，解析其所有一级标签并将其添加到root中，然后对所有一级标签再递归解析出所有子view最终行程一个正确的view树
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 非merge标签处理逻辑
                // 获取到xml布局中的根视图，内部是通过反射来生成对应的view对象
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;
                
                //如果root不为空则解析布局参数，这个布局参数将作用给temp(也就是xml布局中的根视图)，如果root为空则调用方获取到的是一个干净的不带layoutparam的view。
                if (root != null) {
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // 设置布局参数
                        temp.setLayoutParams(params);
                    }
                }

                // xml的根布局生成出来之后就该逐个解析并生成它内部的所有子view以构成一个正确的view树，内部其实是一个递归的过程
                rInflateChildren(parser, temp, attrs, true);

                // 如果root不为空并且调用时attachToRoot指定为true，则意思是需要将view加入root中，do it
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // 从这里可以看出，当调用方并没有指定父容器或者指定了父容器但是不将xml布局加入父容器中，则将xml解析出来的view返回给调用房，否则的话xml解析出来的view其实已经被加入父容器了，所以直接将父容器返回。有时候我们调用inflate时会指定root是非空，attachToRoot是false其实就是为了给xml解析出来的view设置布局参数而已。
                if (root == null || !attachToRoot) {
                    result = temp;
                }
                
            }

        } catch (XmlPullParserException e) {
           ...
        } catch (Exception e) {
            ...
        } finally {
           ...
        }

        return result;
    }
}
```
从方法4实现来看，其主要逻辑是解析xml布局文件并生成对应的view。如果xml是个merge标签的布局则进行merge对应的处理（将一级视图添加到root中以减少一个层级）。**需要注意的是调用inflate时所传入参数不同而获取到的结果就有可能是不同含义：最终如果xml解析出来的布局被成功添加到root中的话则直接返回root容器，否则返回xml解析出来的view。我不知道为什么要这么设计，总感觉这种设计会给开发过程徒增不必要的麻烦**

在我们日常使用中除了`Fragment->onCreateView()`回调中会给我们提供已经创建好的`LayoutInflater`对象之外，我们往往需要自己创建，来看下它的创建过程：

```
mLayoutInflater = LayoutInflater.from(context);

public static LayoutInflater from(@UiContext Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```
LayoutInflater是通过context获取系统服务得到的。

在Activity中往往不需要自己手动创建，因为Activity中已经提供了LayoutInflater对象：

```
Activity.java

@NonNull
public LayoutInflater getLayoutInflater() {
    // 调用PhoneWindow.getLayoutInflater()
    return getWindow().getLayoutInflater();
}


PhoneWindow.java

@UnsupportedAppUsage
public PhoneWindow(Context context) {
    super(context);
    // 此时传入的就是Activity
    mLayoutInflater = LayoutInflater.from(context);
}

@Override
public LayoutInflater getLayoutInflater() {
    return mLayoutInflater;
}

```
在使用`RecyclerView`时经常需要使用到LayoutInflater，设置Adapter时有这么个回调：

```
public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {}
```

看到这里LayoutInflater基本上介绍完了，但是在方法3时有个`tryInflatePrecompiled()`方法顺带提一下
```
private @Nullable
View tryInflatePrecompiled(@LayoutRes int resource, Resources res, @Nullable ViewGroup root,
    boolean attachToRoot) {
    if (!mUseCompiledView) {
        // 未开启预编译则直接返回空，后续会去走xml解析
        return null;
    }
    ...
    // Try to inflate using a precompiled layout.
    String pkg = res.getResourcePackageName(resource);
    String layout = res.getResourceEntryName(resource);

    try {
        Class clazz = Class.forName("" + pkg + ".CompiledView", false, mPrecompiledClassLoader);
        Method inflater = clazz.getMethod(layout, Context.class, int.class);
        // 反射获取到view
        View view = (View) inflater.invoke(null, mContext, resource);

        // 如果root存在，则解析布局参数并且判断是否应该将view添加到root，只有root存在时这个布局参数才有意义，否则直接就是返回一个单纯不带布局参数的view给调用方
        if (view != null && root != null) {
            XmlResourceParser parser = res.getLayout(resource);
            try {
                AttributeSet attrs = Xml.asAttributeSet(parser);
                advanceToRootNode(parser);
                ViewGroup.LayoutParams params = root.generateLayoutParams(attrs);
                if (attachToRoot) {
                    // 将view添加到root容器
                    root.addView(view, params);
                } else {
                    // 给view设置布局参数，此时view还未有任何父容器
                    view.setLayoutParams(params);
                }
            } finally {
                parser.close();
            }
        }

        return view;
    } catch (Throwable e) {
        if (DEBUG) {
            Log.e(TAG, "Failed to use precompiled view", e);
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    return null;
}
```
这是Android10开始的一个编译优化，可将可将xml预编译成dex，然后通过反射生成对应的View，从而减少XmlPullParser解析Xml的时间。截止到目前Android版本33为止预编译这段都是废话，因为追溯`mUseCompiledView`这个开关发现一直都是关着的。即使是打开了也将是个坑，本来inflater的返回值因为root和attachToRoot参数就已经可能产生返回父view和返回xml对应的布局view这两种情况了，而如果加入了预编译选项就不关心root和attachToRoot直接返回xml对应的布局view，更是增加的开发者使用时的不确定性。所以等后续功能放开时再来关注。

### Fragment中使用InflaterLayout报错复盘

知识点介绍完了来看看刚才在`Fragment->onCreateView`中使用`inflater.inflate(R.layout.fragment_me, container)`为什么会报错吧。
首先我们知道了`inflater.inflate(R.layout.fragment_me, container)`的话container不为空，则默认fragment_me对应的布局是需要加入到container中的，此时返回的是父view。此时这个父view正是我们在Activity中指定的Fragment的容器，比如我这边指的是：

```
transaction.add(R.id.f_me, meFragment)//动态添加
```
此时这个f_me已经存在父view了，所以我们看下`Fragment->onCreateView`返回的这个父view在哪里被使用

```
Fragment.java

void performCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
        @Nullable Bundle savedInstanceState) {
    ...
    mView = onCreateView(inflater, container, savedInstanceState);
    ...
}
```
在Fragment源码中，`performCreateView`时会将`onCreateView`的返回值赋值给`mView`对象。

再回到报错的堆栈中`at androidx.fragment.app.FragmentStateManager.addViewToContainer(FragmentStateManager.java:840)`,查看源码可以看出来：
```
void addViewToContainer() {
    int index = mFragmentStore.findFragmentIndexInContainer(mFragment);
    mFragment.mContainer.addView(mFragment.mView, index);
}
```
此时mFragment.mView就是我们在Activity中指定的Fragment容器f_me,它已经存在父容器了所以再被添加的话就报错了！


### LayoutInflater总结
1. infalte()中root为空则返回的是xml布局对应的view，此时view没有任何布局参数，也并未被添加到容器中
2. infalte()中root不为空，则判断attachToRoot==true返回的是root，attachToRoot==false返回的是xml布局对应的view，和第1点一样
3. 如果inflate传递的资源id是个merge标签，则root不能为空，attachToParent必须是true，对应的merge下的view会被添加到root容器中
4. Activity、Fragment、Service中系统已经为你提供了一个LayoutInflater对象，不需自己创建。并且Activity->setContentView()方法最终也是通过InflaterLayout来实现的布局加载
5. 牢记方法2的实现：

```
 public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```
