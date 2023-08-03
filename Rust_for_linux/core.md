本文档的阅读目的

我试图看rust for linux内核的一些东西，但是我发现没有rust核心库的了解是远远不够的

因此开始阅读对此的一个中文注解：https://github.com/Warrenren/inside-rust-std-library



## 概述

内置的intrinsics函数对硬件的封装

traits

self，指代所实现的类型

trait函数，第一个不是参数不是self的任意函数，可以通过trait::function来调用或者实现类型的命名空间来调用。

trait method（方法），第一个参数使用了self，并且是self，&self，&mut self之一的或者被Box,Rc,Arc,Pin包装的

可以用点操作符来调用，或者像函数那样被调用

关联类型

可以使用

```
trait Trait {
    type AssociatedType;
    fn func(arg: Self::AssociatedType);
}
```

这样的方式，在实现的时候定义AssociateedType是啥。

泛型参数

实际包含了泛型类型参数，泛型生命周期参数，泛型常量参数

```
trait Trait<'a, T> {
    // signature uses generic type
    fn func1(arg: T);

    // signature uses lifetime
    fn func2(arg: &'a i32);

    // signature uses generic type & lifetime
    fn func3(arg: &'a T);
}

```

比如这里面就包含了泛型生命周期和泛型类型



关于泛型类型和关联类型

如果对同一个self类型，有单一的实现，（但是不同self类型可以有不同的实现）那么用关联类型

但是对于同一个self类型，有多个实现，那么用泛型类型。

标准库预置

```
use std::prelude::v1::*
```

这里面提供了标准库所实现的一大堆trait，以及部分与他们相同的派生宏

标准库也有一些默认实现

子集和超集问题

（有点像类继承）

例如

```
trait Supertrait {
    fn method(&self) {
        println!("in supertrait");
    }
}

trait Subtrait: Supertrait {
    // this looks like it might impl or
    // override Supertrait::method but it
    // does not
    fn method(&self) {
        println!("in subtrait")
    }
}
```

如果直接写subtrait::method，是不可以的，而要用

```
<SomeType as Supertrait>::method(&st); // prints "in supertrait"
<SomeType as Subtrait>::method(&st); // prints "in subtrait"
```



trait对象

使用dyn trait来表明一个trait对象

```
// trait objects
&dyn Trait
Box<dyn Trait>
Rc<dyn Trait>
Arc<dyn Trait>
```

不是所有trait都能转成trait对象



标记trait

自动trait

unsafe trait，包括Send和Sync



Send和Sync

只有一个类型具有Send trait，才能在线程间安全的发送，只有具有Sync trait，才能在线程间安全的引用。

当且仅当&T是Send，T才能是Sync

Rc不满足Send，需要Arc

在Sync中，则是Rc，Cell，RefCell，需要Mutex来封装

其余都是Sync——原因是在于，除非Arc<Mutex< T >>封装，否则这都是不被允许的。



接下来都是一些具体trait的分析情况了。（不看了）

alloc库

std库（基于操作系统）



rust 可以直接在泛型上面实现trait和方法

T:?Sized是所有的类型， 不带约束的T实际是 T:Sized



Rust的安全特性是建立在一大堆unsafe上面的



## 内存

一个变量的内存参数包括

首地址

占用的内存块大小

内存字节对齐的基数

成员内存顺序



成员的内存顺序完全由编译器控制，而不是C那样的不能变更



裸指针：*const T/*mut T

Rust的裸指针其实不仅仅是一个地址值

两部分组成，第一部分是内存地址，第二部分是对这个地址的约束性描述——元数据

```
//从下面结构定义可以看到，裸指针本质就是PtrComponents<T>
pub(crate) union PtrRepr<T: ?Sized> {
    pub(crate) const_ptr: *const T,
    pub(crate) mut_ptr: *mut T,
    pub(crate) components: PtrComponents<T>,
}

pub(crate) struct PtrComponents<T: ?Sized> {
    //*const ()保证元数据部分是空 
    pub(crate) data_address: *const (),
    //不同类型指针的元数据
    pub(crate) metadata: <T as Pointee>::Metadata,
}

//下面Pointee的定义展示一个RUST的编程技巧，即trait可以只用
//来定义关联类型，Pointee即只用来指定Metadata的类型。
pub trait Pointee {
    /// The type for metadata in pointers and references to `Self`.
    type Metadata: Copy + Send + Sync + Ord + Hash + Unpin;
}
```

瘦指针和胖指针

对于实现了Sized trait的类型（也就是大小确定的），定义为瘦指针，元数据为0

对于动态大小类型则是胖指针

如果是结构类型，只允许最后一个数据类型是动态类型，则元数据是最后一个动态类型的大小的元数据

str类型，按照字节计算大小，为usize

切片类型，数组元素数目，为usize

trait对象，例如 dyn SomeTrait， 元数据是 [DynMetadata][DynMetadata]

对于trait对象的元数据。

```
//dyn trait裸指针的元数据结构,此元数据会被用于获取trait的方法
pub struct DynMetadata<Dyn: ?Sized> {
    //在堆内存中的VTTable变量的引用,VTable见后面的说明
    vtable_ptr: &'static VTable,
    //标示结构对Dyn的所有权关系，
    //其中PhantomData与具体变量的联系在初始化时由编译器自行推断完成, 
    //这里PhantomData主要对编译器提示做Drop check时注意本结构体会
    //负责对Dyn类型变量做drop。
    phantom: crate::marker::PhantomData<Dyn>,
}

//此结构是实际的trait实现
struct VTable {
    //trait对象的drop方法的指针
    drop_in_place: fn(*mut ()),
    //trait对象的内存大小
    size_of: usize,
    //trait对象的内存对齐
    align_of: usize,
    //后继是trait对象的所有方法的指针数组
}
```

元数据类型相同的裸指针可以任意切换，但是元数据不同的裸指针不能任意切换。



MaybeUnInit< T >

使用这个封装了的T，如果没有初始化就不会调用drop，从而避免引起一大堆问题。

```
 #[repr(transparent)] 
    pub union MaybeUninit<T> {
        uninit: (),
        value: ManuallyDrop<T>,
    }
 #[repr(transparent)]
    pub struct ManuallyDrop<T: ?Sized> {
        value: T,
    }


```

MaybeUnInit内存布局就是ManuallyDrop内存布局，ManuallyDrop内存布局就是T内存布局，所以从内存上而言，这仨是一样的

对于MaybeUninit的使用应该是

MaybeUninit::< T >::uninit()

而不是MaybeUninit< T >::new(val:T)，这里是有用val进行初始化的

MaybeUninit< T >::zeroed()，全都弄成0



对一个未初始化的对象写入值：

MaybeUninit< T >::write(val)

返回值是一个&mut T

MaybeUninit< T >::assume_init()->T

此时，MaybeUninit这个东西已经被消费掉了

如果一直都没有转移所有权，因为这玩意儿不会自动释放，因此调用MaybeUninit< T >::assume_init_drop(&self)

或者手工使用drop_in_place进行释放

```
ptr::read< T >(src: *const T) -> T    //浅拷贝
ptr::read_unaligned<T>(src: *const T) -> T    //如果不确定内存是否对齐，需要用这个
ptr::write<T>(dst: *mut T, src: T)
ptr::write_unaligned<T>(dst: *mut T, src: T)
```

&必须要求字节对齐

&raw则不要求这一点



NonNull< T >

```
#[repr(transparent)]
pub struct NonNull<T: ?Sized> {
    pointer: *const T,
}
```

Unique< T >

```
#[repr(transparent)]
pub struct Unique<T: ?Sized> {
    pointer: *const T,
    _marker: PhantomData<T>,
}
```

和上面一对比就知道，多了个_marker: PhantomData< T >，作用是告诉编译器他具有所有权，从而能够实现Send，Sync等等需要所有权的特性，Unique指针被封装前，必须保证他不是NonNull< T >

同样的道理，为了保证具有堆空间申请内存的所有权，必须要用Unique < T >



Mem模块函数

`mem::zeroed<T>() -> T` 返回一个内存块清零的泛型变量，内存块在栈空间

`mem::uninitialized<T>() -> T` 返回一个未初始化过的泛型变量，内存块在栈空间



Layout

```
pub struct Layout {
    // 类型需占用的内存大小，用字节数目表示
    size_: usize,
    //  按照此字节数目进行类型内存对齐， 
    //  NonZeroUsize见代码后面文字分析
    align_: NonZeroUsize,
}
```

也就是这个对象的大小和对齐情况。
