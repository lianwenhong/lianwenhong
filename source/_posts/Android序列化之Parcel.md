---
title: Android序列化之Parcel
date: 2022-06-07 19:41:11
tags: binder
categories: Android
---
Android操作系统的底层数据传输形式是简单的字节序列形式进行传递。用通俗的话说就是系统不认识对象，只认识字节序列。而我们为了达到通信或者存储的目的，需要先将数据序列化传递，要使用时再进行反序列化还原。

### Android中有2种方式可以实现序列化和反序列化：
1. Serializable：Java自带的接口，实现该接口就可以将对象序列化。
2. Parcelable：Android独有的接口，性能优于Serializable。原理是将一个完整的对象进行分解(拍扁)，分解后的每一部分都是Intent所支持的数据类型。

#### 二者区别是什么呢？
- Serializable使用IO读写将序列化对象存储在硬盘上，读写速度慢；序列化过程中使用了反射技术所以会产生很多临时对象，占用空间大。但是它的优点是编码方便，开发者只需要实现Serializeable，对象就拥有了序列化和反序列化能力。
- Parcelable是直接再内存中进行读写，内存读写速度优于硬盘读写，所以这种方式的性能比Serializable高，缺点是编码比Serializable方式更麻烦。

#### 使用选择
- 如果仅仅是在内存中使用，比如Activity、Service间传递信息，那强烈建议使用Parcelable，因为Parcelable比Serializable性能高，并且Serializable在序列化时会产生大量临时变量从而引起频繁GC。
- 如果是持久化操作，推荐使用Serializable，虽然效率比较低但是因为再外界有变化的情况下，Parcelable不能很好的保存数据的持续性。

#### 为什么Android中进程之间的复杂数据类型传递需要序列化
我们知道Android系统是基于Linux系统实现的，而Linux有进程隔离的机制。而进程如果传递复杂数据类型那传递的是对象的引用，本质上就是一个内存地址。但是传递内存地址的方式在跨进程中明显不行，由于Linux采用了虚拟内存机制，两个进程都有自己独立的内存地址空间，所以把A进程中某个对象的内存地址传递给B进程，这个内存地址在两个进程中映射到的物理内存地址并不是同一个，所以就得依靠上述的序列化手段来将对象的字节序列传递才能实现通信。而对于基本数据类型，只需要通过IPC通信不断复制达到目标进程即可。

#### Parcel用于IPC
> Container for a message (data and object references) that can be sent through an IBinder. A Parcel can
contain both flattened data that will be unflattened on the other side of the IPC (using the various
methods here for writing specific types, or the general Parcelable interface), and references to
live IBinder objects that will result in the other side receiving a proxy IBinder connected with the
original IBinder in the Parcel.
Parcel is not a general-purpose serialization mechanism. This class (and the corresponding Parcelable API
for placing arbitrary objects into a Parcel) is designed as a high-performance IPC transport. As such,
it is not appropriate to place any Parcel data in to persistent storage: changes in the underlying
implementation of any of the data in the Parcel can render older data unreadable.

Parcel是盛放消息的容器，是接住Binder机制来进行数据传输的。它可以携带序列化后的数据通过IPC传输后在目的端进行反序列化。

Parcel也可以传递IBinder对象，在目的端将接受到传输的IBinder对象的代理。

Parcel不适合用于持久性存储，因为Parcel中任何数据的基础实现的更改都可能导致旧值不可用。并且其是在内存中实现，内存中持久化数据不可靠。

{% asset_img Parcel进程间传递数据.jpg Parcel进程间传递数据 %}

- 从Android系统层面来说，Android系统中的Binder机制实现的IPC就使用Parcel类来进行客户端与服务端的数据交互。并且在Java层和cpp层都实现了Parcel（通过JNI关联）。由于它在c/cpp中直接使用了内存来读取数据，因此效率更高。
- 从存储的角度来说，Parcel只是一块连续的内存。会根据需要自动扩展大小。

Parcel传输数据类型：
> **基本数据类型：** 借助Parcel->writePrimitives()将基本数据类型从用户空间(源进程)copy到kernel空间(Binder驱动中)再写回用户空间(目标进程，binder驱动负责寻找目标进程)

> **复杂数据类型：** 将经过序列化的数据借助Parcel->writeParcelable()/writeSerializable()从用户空间(源进程)copy到kernel空间(binder驱动中)再写回用户空间(目标进程，binder驱动负责寻找目标进程)，然后再进行反序列化。

> **大数据：** 通过Parcel->writeFileDescriptor()通过Binder传递匿名共享内存(Ashmem)的FileDescriptor从而达到传递匿名共享内存的方式，即传递的是FileDescriptor而不是真正的大数据。参考[Android 匿名共享内存的使用](https://zhuanlan.zhihu.com/p/92769131)

> **IBinder对象：** 通过调用Parcel->writeStrongBinder()，经由kernel binder驱动专门处理来完成IBinder传递。目标进程收到的是IBinder对象的代理。