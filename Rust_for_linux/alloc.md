# 前言

对于操作系统而言，alloc部分是最复杂的，在rust for linux中，同样这部分也是最复杂的一部分。

本文档所基于的rust for linux版本号，是基于2023/8/1号的最新分支

本文档涉及的文件包括

rust/alloc/

rust/kernel/allocator.rs

rust/kernel/mm.rs

rust/kernel/pages.rs

这么几个文件

按照实际而言，我从底向上进行描述

# Kernel/pages.rs

按照注释而言，该模块目前仍在完善过程中

### 该模块主要实现的内容包括：

1、struct Pages：

实现的函数有：

1. new方法，对alloc_pages进行了封装

2. copy_into_pages方法，将给出的UserSlicePtrReader（注：见user_ptr模块）拷贝到对应的page里面，但是目前仅仅完成了第一个页，也就是说拷贝的内容不能大于一个页面大小

3. read方法，同样只包括了第一个页面，封装了ptr::copy（这应该看core库）

4. write方法，同样只包含第一个页面，同样用ptr::copy。

5. kmap方法，使用的是kmap来进行映射，返回的是下面封装的Option< struct PageMapping >，

除此之外，为struct Pages实现了Drop trait，封装了__free_pages

2、struct PageMapping

仅仅实现了对page的封装，然后实现了Drop trait，封装了kunmap

# Kernel/mm.rs

该模块调用了上面的pages模块

### 该模块对应的Linux模块为

include/linux/mm.h

但是其实现的内容页非常有限

### 该模块做的事情包括

对VM的flags进行了封装整理

对vm_area_struct进行了封装成了rust中的Area结构体，该结构体描述了一个virtual memory area。

对该Area结构体，实现了以下几个方法：

1. from_ptr方法，反正就是一个ptr转过来的封装呗

2. flags方法，返回flags

3. set_flags方法，设置flags

4. start方法，得到start

5. end方法，得到end

6. insert pages方法，封装了vm_insert_page，将一页插入到这个vm结构体中去。

### 个人认为需要完善的地方

在mm.h中有个vm_operations_struct

里面封装了对vm area的一些操作，以及后面也有大量的操作仍然需要封装。我不认为这些都一定是必须的，但是，需要有。



# kernel/allocator.rs

（注：no_mangle，用于关闭rust修改名称功能，使得最后生成的函数名称，在链接器中，就是我们写的名称）

定义了一个KernelAllocator结构体，并且将其设置为一个全局变量

为这个KernelAllocator实现了GlobalAlloc trait（里面仅仅包含alloc和dealloc两个方法），该trait是Rust公共的一个接口

这两个方法分别封装了krealloc和kfree两个Linux中的调用。



后面用

```
#[allow(clippy::no_mangle_with_rust_abi)]
#[no_mangle]
```

定义了四个函数

- __rust_alloc

- __rust_dealloc

- __rust_realloc

- __rust_alloc_zeroed

同样是对krealloc和kfree两个Linux中的调用的封装。



关于GlobalAlloc这个trait

后续对这部分函数的封装的工作在alloc模块中体现出来，在此不做赘述。

但总之，这是rust为程序员手动实现的内存分配器实现的框架。

在Linux中，有非常多的用于分配内核内存的其他函数，而这边仅仅使用了个位数的数量，但是，我个人认为，也不是不能用。



# alloc/文件夹

## alloc/alloc.rs

接着上面的allocator.rs文件，首先将上述__rust_这几个函数声明为extern ”rust“，用于引用他们。

然后会将上述的__rust_这几个函数再次封装成

alloc，dealloc，realloc，alloc_zeroed四个函数

```
extern "Rust" {
    #[rustc_allocator]
    #[rustc_allocator_nounwind]
    fn __rust_alloc(size: usize, align: usize) -> *mut u8;
    #[rustc_allocator_nounwind]
    fn __rust_dealloc(ptr: *mut u8, size: usize, align: usize);
    #[rustc_allocator_nounwind]
    fn __rust_realloc(ptr: *mut u8, old_size: usize, align: usize, new_size: usize) -> *mut u8;
    #[rustc_allocator_nounwind]
    fn __rust_alloc_zeroed(size: usize, align: usize) -> *mut u8;
}

//对__rust_xxxxx_再次封装
pub unsafe fn alloc(layout: Layout) -> *mut u8 {
    unsafe { __rust_alloc(layout.size(), layout.align()) }
}

pub unsafe fn dealloc(ptr: *mut u8, layout: Layout) {
    unsafe { __rust_dealloc(ptr, layout.size(), layout.align()) }
}

pub unsafe fn realloc(ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
    unsafe { __rust_realloc(ptr, layout.size(), layout.align(), new_size) }
}

pub unsafe fn alloc_zeroed(layout: Layout) -> *mut u8 {
    unsafe { __rust_alloc_zeroed(layout.size(), layout.align()) }
}
```

在此基础上，为Global实现了alloc_impl和grow_impl两个方法，以及Allocator的trait。

```
impl Global{
    #[inline]
    fn alloc_impl(&self,layout:Layout,zeroed:bool) -> Result<NonNull<[u8]>,AllocError>
    #[inline]
    unsafe fn grow_impl(&self,ptr:NonNull<u8>,old_layout:Layout,new_layout:Layout,zeroed:bool) -> Result<NonNull<[u8]>,AllocError>
}
impl Allocator for Global{
    fn allocate
    fn allocate_zeroed
    unsafe fn deallocate
    unsafe fn grow
    unsafe fn grow_zeroed
    unsafe fn shrink
}
```

这个Allocator的trait的实现，则是在alloc_impl和grow_impl方法以及上面四个函数的基础上实现的，当然像shink这样的，也有调用Allocator这个trait自己的其他方法。

最终实现的就是Allocator的这些方法。



在alloc.rs这个文件中，还有其他内容。

- exchange_malloc，调用了global的allocate，并且用了后续的handle_alloc_error

- box_free，堆分配器使用不同的allcoator的封装

- __rust_alloc_error_handler并封装成handle_alloc_error

- WriteCloneIntoRaw，这是在后续的Box::clone和Rc/Arc::make_mut被使用到的

总之，这里面alloc.rs模块，基本上是alloc文件夹下面其他模块都公用的一个部分了，handle_alloc_error、Allocator、Global这几个都会被用到。



## alloc/Borrow.rs

Borrow trait是内部可变性的基础

第一部分是Borrow和ToOwned trait的实现

```
pub trait ToOwned{
    type Owned:Borrow<Self>;
    fn to_owned(&self) -> Self:Owned;
    fn clone_into(&self,target:&mut Self:Owned)
    {
        *target = self.to_owned()
    }
}
impl<T> ToOwned for T
where
    T:Clone
...
```

但是另一部分则是Cow（clone on write）

```
pub enum Cow<'a, B: ?Sized+ 'a>
where
    B:ToOwned,
{
    Borrowed,
    Owned
}
```

以及在此基础上实现了对于Cow枚举的几个trait和方法

```
impl Cow{
    pub const fn is_borrowed(&self)-> bool
    pub const fn is_owned(&self) -> bool
    pub fn to_mut(&mut self) -> &mut <B as ToOwned>::Owned
    pub fn into_owned(self) -> <B as ToOwned>::Owned
}
impl Borrow for Cow
impl Clone for Cow
impl Deref for Cow
impl Eq for Cow
impl Ord for Cow
impl PartialEq for Cow
impl PartialOrd for Cow
impl fmt:Display for Cow
impl fmt:Debug for Cow
impl Default for Cow
impl AsRef<T> for Cow
impl Add<&'a str> for Cow
impl Add<Cow<'a,str>> for Cow
impl AddAssign<&'a str> for Cow
impl AddAssign<Cow<'a,str>> for Cow
```

实现内容非常扁平化，就是对这几个trait的实现，没啥复杂结构

关于这部分的具体含义，自行了解。

## alloc/collections

这个文件夹下只有一个mod.rs，该文件包含的内容如下：

引用了btree，binary_heap，linked_list，vec_deque等高级数据结构

定义了TryReserveErrorKind的枚举，并以TryReserveError结构体封装。

```
pub struct TryReserveError{
    kind: TryReserveErrorKind,
}
pub enum TryReserveErrorKind{
    CapacityOverflow,
    AllocError,
}
```

并且为这些错误类型实现了类型转换以及Display等等常见功能。

```
impl From<TryReserveErrorKind> for TryReserveError
impl From<LayoutError> for TryReserverError
impl Display for TryReserveError
......
```

## alloc/raw_vec.rs及alloc/Vec/文件夹

要说Vec的实现，必须先从raw_vec.rs说起。

在raw_vec.rs中，定义了RawVec结构体

```
pub(crate) struct RawVec<T,A:Allocator = Global>{
    ptr:Unique<T>,
    cap:usize,
    alloc:A,
}
```

包含了一个指针，一个容量，一个分配器。

然后为这个RawVec实现了一大堆，唔，我感觉跟C的这个Vec差不多的东西，什么扩容缩容之类的，啊反正上过数据结构肯定知道了，这些方法比较类似，因此不在此列举所有的函数方法。

以及为上述这些RawVec方法实现的一些工具函数以便于操作。



对于Vec文件夹

从mod.rs 开始看起

### alloc/vec/mod.rs

对该文件夹下面的其他模块进行引用，定义了Vec结构体

```
pub struct Vec<T, #[unstable(feature)="allocator_api",issue="32838"] A:Allocator=Global>{
    buf: RawVec<T,A>,
    len: usize,
}
```

该结构体对RawVec进行了封装，封装成了Vec

整个文件提供了对于Vec所有的接口，实在太多，我不一一列举。

相比于slice，into_itor和drain是vec特有的东西。

### alloc/vec/into_iter.rs

虽然实际文件顺序是drain，drain_filter之后才是into_iter，但是还是得先讲这个。

into_iter的主要作用是以值而不是引用的方式访问

结构体如下

```
pub struct IntoIter< T , A : Allocator = Global >
{
    pub(super) buf: NonNull<T>,
    pub(super) phantom: PhantomData<T>,
    pub(super) cap: usize,
    pub(super) alloc: ManuallyDrop<A>,
    pub(super) ptr: *const T,
    pub(super) end: *const T,
}
```

对于这个NonNull后面还要跟phantomdata，我其实有个疑问，他为什么不用Unique< T >？

 IntoIter实现的方法：

有进行转换的as_slice，as_mut_slice，as_raw_mut_slice，into_vecdeque。

有对于drop的封装的forget_allocation_drop_remaining,forget_remaining_elements。

IntoIter实现的trait：

AsRef，Send，Sync，Iterator，DoubleEndediterator（Iterator是前向迭代，而DoubleEndediterator是逆向迭代），ExactSizeIterator，FusedIterator，TrustedLen，intoIter，Clone，Drop，SourceIter，AsVecIntoIter

这些都是封装好的接口trait，语义基本一致，没啥可以多说的。

### alloc/vec/drain.rs



### alloc/vec/drain_filter.rs



### alloc/vec/is_zero.rs

这个文件里面的一些实现细节值得推敲，尤其是对于不同类型的宏实现，非常有意思。

impl_is_zero！宏来判断一些常用类型是否为0

使用IsZero trait来判断裸指针类型，切片类型，元组类型等等是否为0。

总而言之，整个文件里面主要做的事情就是对于不同的类型来判断是否为0



### alloc/vec/partial_eq.rs

主要实现了一个__impl_slice_eq1的宏，在其中实现了PartialEq的trait，包括eq和ne两个函数

然后为各种Vec的封装情况实现了这个比较。

整个文件的作用是字面意思的。

### alloc/vec/set_len_on_drop.rs

按照注释的说法，这是为了避免别名分析所带来的问题。

说是length字段是一个local变量，从而不会去跟Vec中的len去进行对比优化

在这里实现了一个结构体SetLenOnDrop

```
pub(crate) struct SetLenOnDrop<'a>
{
    len: &'a mut usize,
    local_len:usize,
}
```

并且为其实现了new，increment_len ，current_len，drop等等操作

只有在drop的时候，才让len=local_len。

### alloc/vec/spec_extend.rs

这个文件为各种vec的情况实现了两个trait

```
pub(super) trait SpecExtend{}
pub(super) trait TrySpecExtend{}
```

这两个trait的作用是，vec::extend和vec::tryextend的specialization版本。



总结，总体而言，使用raw_vec和vec的封装，基本完成了rust中vec的操作。



## alloc/fmt，alloc/slice，alloc/str，alloc/string以及alloc/ffi/c_str等字符串相关

### fmt.rs

本文件的内容非常简单，封装使用了核心库的fmt，以及实现了一个format函数，该函数会在marcos.rs中被调用，反正就是用于格式化输出字符串的，类似于printf/fprintf。

但是有一大堆注释用于说明，有兴趣的可以去看看。在注释中，除了本身一大堆的format之外，还包括了对于一个实现format输出的示例。示例了fmt::Display和fmt::Debug的实现和差异。

### slice.rs



### string.rs



### alloc/str.rs



### kernel/str.rs

值得注意的是，在kernel文件夹下面也定义了str.rs。

首先定义了CStrConvertError类型并把它和Error的转换做掉。

然后实现了CStr结构体，在这个结构体对应impl里面，实现了比如获取长度，各种转换等等功能

接下来是fmt的部分，确切的说是，为CStr实现了fmt的Display和Debug trait

接下来是为CStr实现一些其他的trait，比如AsRef，Deref，Index< ops::RangeFrom >，Index< ops::RangeFull >

(我觉得这些内容无需多言，有rust相关基础的应该很好理解做了什么)

c_str宏和test（这无关紧要）

RawFormatter



### ffi/c_str.rs

值得注意的是，ffi里面迄今为止仍然只有c_str这一部分，所以那个mod.rs仍然就是挂羊头卖狗肉的。

但是，无论是从ffi这个文件夹命名，还是从mod.rs这个文件存在来讲，这个文件夹下面，在设计中，应该还有一些需要C代码ABI的情况下，需要进行转换的，虽然mod.rs很常见，但是比如下面的boxed里面就没有提供，而是直接裸的一个thin.rs。不过在mod.rs中的注释里面又明确的指明了是针对于Cstring而言的。

我个人对此的猜测是，给其他C风格的模块的扩充提供了可能

根据mod.rs中的注释，rust和c的string的区别主要在于以下几点：

- 编码方式：rust是UTF-8，C则不同

- 符号size：C使用char或者wchar_t，而rust是UTF-8

- 没有终止符或者隐含的string length：c有个\0作为终止符，然后去计算长度，但是rust的string存储了长度，所以不用计算

- 在字符串中间的终止符，C的字符串中间不可能有\0的，但是rust没所谓
  
  

总结：在上述的这几个模块中，总体实现了对于rust中字符串的处理。



## alloc/Boxed/thin.rs和alloc/Boxed.rs

Boxed.rs引用了上述的大部分的模块。

对于thin.rs，首先必须了解rust中瘦指针和胖指针的区别

对于实现了Sized trait，具有固定大小的对象的指针，其元数据大小为0，对于动态类型，元数据大小不为0的，则是胖指针。

所以，字面意义而言，thin.rs仅仅完成了瘦指针相关的内容。



## alloc/macros.rs

实现了几个宏

```
vec!
format!
__rust_force_expr!
```

## alloc/lib.rs

这个模块就跟一般的lib.rs没啥区别了，就是一个对外的接口了。包含了这个模块内的其他所有模块



# 总结

关于这部分模块的内容，主要是按照rust所给定的框架，实现了alloc，以及一些rust内部基本数据结构在Linux环境下的实现。

总体而言，除了kernel/alloc等文件中调用了bindings来调用Linux的实现，其他没啥太多差别，alloc本身也是作为rust基本内容来实现的

另外，在rust for linux中，很多涉及到linux的部分的接口bindings缺少非常多，但是，alloc这部分，更多的是偏向于rust语言层面的，所以除了在kernel中的几个文件对linux的接口进行了部分封装（并且我也在上面明确列举出来），其他文件并不存在需要补充完善的地方。

事实上，很多文件其实本来在rust官方的alloc crate中就有，也是相对比较标准的库文件了，但是为了能放进rust for linux，这部分可以看出来做了一定的适配，事实上，std库便是如此，是一个依赖于操作系统的库，不过这里已经没必要实现std库而是自己搭建了一个就是了。


