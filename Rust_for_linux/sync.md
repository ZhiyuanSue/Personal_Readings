# Introduction

Rust for Linux在同步方面的内容主要放在以下几个文件及目录中

kernel/sync.rs

kernel/async.rs

kernel/sync/

kernel/kasync/

从字面上也好理解，async是异步，而sync是同步



# SYNC

## sync.rs

在这个文件中，引用了sync/文件夹中的所有模块中的对象。

除此之外，在这个文件中实现了LockClassKey结构体。

```
pub struct LockClassKey(UnsafeCell<MaybeUninit<bindings::lock_class_key>>);
```

对于这里直接绑定的lock_class_key，其C代码的实现在lockdep_types.h：

```
struct lock_class_key{
    union{
        struct hlist_node hash_entry;
        struct lockdep_subclass_key subkeys[MAX_LOCKDEP_SUBCLASSES];
    };
}
struct lockdep_subclass_key{
    char __one_byte;
} __attribute__ ((__packed__));
```

关于这个的具体作用，看代码和注释，感觉像是给每个锁一个hash编号，如果是static的锁，直接用地址。

并为他实现了一个new和get的操作

以及实现了一个trait NeedsLockClass

```
pub trait NeedsLockClass{
    fn init(...);
}
```

在此基础上，实现了两个宏init_with_lockdep以及init_static_sync

init_static_sync这个宏最终还是调用了init_with_lockdep

init_with_lockdep宏的作用就是给传入的obj加上这个lock必要的一些东西，比如hash值和name什么的。

由于这里只有NeedsLockClass这个trait的定义，所以具体到某个类型，比如SpinLock和Mutex还是得自己实现一个NeedsLockClass这个trait。



## sync/

### guard.rs

虽然按照字母顺序应该是先描述arc.rs，但是guard.rs是整个sync的一个基础。

他定义了一堆trait，而这些trait会被其他模块所调用并实现。

不按照顺序来，我认为首先应该描述的是LockInfo这个trait

```
pub trait LockInfo{
    type Writable:Bool;
}
```

在这个基础上实现了ReadLock和WriteLock两个结构体

```
pub struct ReadLock;
impl LockInfo for ReadLock{
    type Writable=False;
}
pub struct Writable;
impl LockInfo for WriteLock{
    type Writable=True;
}
```

在这两个trait基础上实现了Lock trait（用LockInfo是ReadLock还是WriteLock来区分Lock的属性）

```
pub unsafe trait Lock<I:LockInfo = WriteLock>{
    type Inner: ?Sized;
    type GuardContext;

    fn lock_noguard
    fn relock
    unsafe fn unlock
    fn locked_data
}
```

在Lock这个trait的基础上，实现了Guard。

```
pub struct Guard<'a, L:Lock<I> + ?Sized , I:LockInfo = WriteLock>{
    pub(crate) lock: &'a L,
    pub(crate) context: L::GuardContext,
}
```

并为Guard实现了如下的trait

- Sync

- Deref

- DerefMut

- Drop

对于上述的结构，最主要的还是Guard这个结构体，他存在的意义在于给任何实现了Lock trait的结构体一个Lock guard。



除此之外，还额外实现了两个trait：

```
pub trait LockFactory{
    type LockedType<T>;
    unsafe fn new_lock<T>(data:T)->Self::LockedType<T>;
}
pub trait LockIniter{
    fn init_lock
}
```

以及承接上面的sync.rs的内容，为LockIniter实现了NeedsLockClass的具体实现。

对于这两个trait

LockFactory按照注释是实现一个具体的实例

而LockIniter在我看来就是为了NeedsLockClass的初始化而实现的。

### arc.rs



### condvar.rs



### locked_by.rs



### mutex.rs



### nowait.rs



### rcu.rs



### revocable.rs



### rwsem.rs



### seqlock.rs



### smutex.rs



### spinlock.rs

spinlock.rs对应的头文件为include/linux/spinlock.h

在本文件中实现的内容包括RawSpinLock和SpinLock

关于SpinLock的结构体定义如下

```
pub struct SpinLock<T:?Sized>{
    spin_lock: Opaque<bindings::spinlock>,
    _pin:PhantomPinned,
    data: UnsafeCell<T>,
}
```

在这里使用了一个Opaque的类型，该类型定义在types.rs中

按照注释的说法，这是在FFI对象中，不进行Rust的任何转换

```
pub struct Opaque<T>(MaybeUninit<UnsafeCell<T>>);
```

个人看法，不就是对MaybeUninit的一个封装嘛。

关于RawSpinLock的结构体的定义如下：

```
pub struct RawSpinLock<T: ?Sized>{
    spin_lock: Opaque<bindings::raw_spinlock>,
    _pin:PhantomPinned,
    data: UnsafeCell<T>,
}
```

感觉和SpinLock没啥差别，就是封装的对象不一样罢了。





除此之外，实现了spinlock_init，调用了sync.rs中的init_with_lockdep宏，具体做了什么上面已经讲了。
