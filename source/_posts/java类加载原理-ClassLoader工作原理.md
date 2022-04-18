---
title: java类加载原理(ClassLoader工作原理)
date: 2022-04-15 18:51:47
tags:
categories: Java
---
## 什么是类加载
在java开发中我们在.java文件中编写代码，之后通过javac指令编译成.class文件。当然如果是编写的Android应用后续又会经过dx工具将.class变为.dex文件(这不是本章重点)。

当javac编译成.class文件之后，它是以文件的形式存在的。比如MainActivity.java -> javac -> MainActivity.class。当我们程序中要使用MainActivity的时候，肯定要将这个文件加载到内存中来变成一个类对象这样才能去使用它。

<!-- more -->

{% asset_img 类加载原理.png 类加载原理 %}

**具体来说类加载就是将类的.class文件转为二进制数据读取到虚拟机内存的方法区中，当对象实例进行创建的时候会在堆内存中创建这个类的class对象，用来封装类在方法区中的数据结构。**

**类加载最终的产物就是这个类对应的Class对象，它封装了方法区中的类的数据结构，并向程序员提供了访问接口。**

## 类加载过程
java的类加载分为3个步骤：
1. 装载（Load)：查找并加载类的二进制字节码(.class)文件。
2. 链接（Link）
3. 初始化（Initialize）：**对类的静态变量、静态代码块执行初始化操作**。

对于链接这一过程分为3个子过程：
1. 验证：确保被加载的类的正确性
2. 准备：为类的静态变量分配内存并初始化为默认值（这个初始化与初始化流程中的初始化不一样，这里是初始化为默认值，下一个流程中是初始化为指定值）
3. 解析：把类中的符号引用转换为直接引用

{% asset_img java类加载过程.jpg java类加载过程 %}

#### 思考1:为什么要有验证阶段？
由Java编译器编译生成的.class文件一定是符合JVM字节码格式的，由JVM加载并运行，为了避免人为写一个.class文件，用于恶意用途，就需要有验证，不符合格式的不让它继续运行，确保安全。
#### 思考2:链接中的准备过程和初始化过程有什么区别？
准备过程是初始化为默认值，而初始化过程是将变量赋予正确的值。
#### 什么情况会导致类的初始化？
- 创建类实例时（也就是new）
- 访问某个类或者接口的静态变量时（其实接口中的变量都是public final static的），或者是对该静态变量进行赋值时
- 调用类的静态方法
- 反射时（Class.forName(clzName)），这种情况下面会有说明
- 初始化一个类的子类(会首先初始化其父类)
- JVM启动时表明的启动类，即文件名和类名相同的那个类

#### 系统为我们提供了2种类加载方式：
- Class.forName(clzName)
- ClassLoader.loadClass(clzName)

#### 类加载器
类加载器是系统为我们提供的加载类的工具类，基类是ClassLoader，在Java中默认提供了：
- BootStrap:引导类加载器（启动类加载器）
- ExtClassLoader:拓展类加载器
- AppClassLoader：系统类加载器
  除了这几个默认的之外我们也可以自定义一个类加载器去加载指定目录中的类。例如Android中定义了：
- BootClassLoader:加载AndroidFramework层class文件（Android系统类）
- BaseDexClassLoader(其子类有PathClassLoader、DexClassLoader用于加载指定目录虾的.dex文件中的class类)

**归根结底其实类加载器就是一个加载类的工具，而不同类加载器的区别就是加载的类所处的目录不同而已**，具体原因后续讲双亲委托机制时详述。

#### 思考3:类加载也是类，那类加载器被谁加载？
BootStrap不是java类，它是c/c++写的，是jvm内核的组成部分，它主要用于加载jvm运行必须的类。而ExtClassLoader和AppClassLoadeer是java类，他们是被BootStrap所加载。BootStrap是无法被java程序直接引用的。

## 双亲委托机制
{% asset_img 双亲委托机制.jpg 双亲委托机制 %}

#### 类加载器的父子结构
jvm所有的类加载器都采用父子关系的树形结构进行组织（但是注意不是java语法上的父子关系，不是继承），它的原理是在实例化每个类加载器的时候都得为其指定一个父类加载器对象或者默认采用系统类加载器为其父类加载器。（类加载器中有一个parent属性，组织父子结构是通过为parent赋值来实现的）

每个类加载器都有自己的加载区域，它只能在自己的加载区域中寻找类。

#### 双亲委托机制工作原理
**当使用某个类加载器去加载一个类的时候，首先会委托给其父类去加载，当父类还有父类的时候会一层层向上委托直至没有父类。也就是说父类相对于子类来说具有绝对的优先加载权。因此，所有的类加载请求都应该被传递到顶层的引导类加载器中，如果父类加载器加载到类了那就直接返回该类，只有当父类加载器在它的加载区域中找不到所要加载的类时，才会把加载权再归还给子类去加载。**

这就是大名鼎鼎的双亲委托机制。

这么设计有什么好处呢？

1. 保证了同一个类不会被重复加载（不会被多个类加载器加载）
2. 保证了类加载的安全性（例如假如有人自己创建了一个java.lang.String类也无法替换掉系统中的java.lang.String类，因为类加载会最终委托到父类加载器中，所以系统中的java.lang.String一定是先被找到的）

**类加载器加载一个类的时候会首先从缓存中找，如果找到了就直接返回类，如果没找到才会继续向上委托。当成功加载到某个类的时候，会将得到的Class对象缓存起来用于下次加载这个类的时候直接返回。**
不信可以看看java ClassLoader类的loadClass()实现：

```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 找缓存
        Class<?> c = findLoadedClass(name);
        // 缓存中没有，则开始双亲委托机制
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 从父类加载器加载
                    c = parent.loadClass(name, false);
                } else {
                    // 没有父说明这个已经是Bootstrap
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            
            // 没有加载到，则在自己的区域中寻找类
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

#### 思考4:为什么静态元素和静态代码块在虚拟机种只会执行一次？
因为默认情况下赋值静态域的过程（初始化）或者是执行静态代码块是在类加载流程中执行的，而同一个类根据双亲委托机制只会被加载一次，如果第二次加载它会直接从缓存中取。当然如果你自己定义了一个类加载器再去手动加载它的话就另当别论了（假设加载类A，正常代码中类在AppClassLoader中被加载成功，此时你自定义一个类加载器并且它的加载区域中也有类A一样的class，并且把它的父类加载器设置成ExtClassLoader，那么肯定这时候通过自定义类加载器去加载那也能加载出一个类A，此时类A中的静态代码也会执行，**但是，但是，但是这个但是很重要，但是此时加载出来的这俩类A不是同一个类**）。

#### 届定2个类是不是同一个类对象
**这也就是思考4中的但是，2个类对象是否是同一个取决于：类的全类名、加载它的类加载器以及类加载器的实例这三个条件，这三个条件中只要有一个不相同，加载后生成的Class对象就不是同一个，最终生成的类的实例的类型也不相同。**

所以即使是同一个类，用不同的类加载器去加载的话产生的对象也是不同的，不能进行互相赋值否则会出现类转换异常。

#### Class.forName(clzName)的原理
直接上代码：

```
public static Class<?> forName(String className)
            throws ClassNotFoundException {
    Class<?> caller = Reflection.getCallerClass();
    // 内部也是使用ClassLoader进行类加载，只是加载完成之后会自动执行初始化，同时forName0()的initialize参数决定
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}

private static native Class<?> forName0(String name, boolean initialize,
                                    ClassLoader loader,
                                    Class<?> caller)
throws ClassNotFoundException;
```
所以Class.forName(clzName)本质上也是通过ClassLoader.loadClass(clzName)进行类加载，只是它会执行初始化，而直接通过ClassLoader.loadClass(clzName) 加载的类只有达到上面说的几个条件才会执行初始化流程。

下一章我们讲一下Android中的类加载器：[Android源码解析-App的ClassLoader](http://lianwenhong.top/2022/04/14/Android%E6%BA%90%E7%A0%81%E5%B0%8F%E8%AE%B0-App%E7%9A%84ClassLoader/#more)