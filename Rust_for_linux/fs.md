# Introduction

对于文件系统部分

我认为其包括了几个部分

kernel/fs/

kernel/file.rs

kernel/fs.rs

与alloc部分不同，alloc主要偏向于rust的内置数据结构的实现，与linux关系不大，但是文件系统部分，会有更多内容与linux的文件系统相联系起来，因此，除了对上述文件的分析，还将与linux文件系统对应的内容进行大量的对比。

# kernel/file.rs

按照注释所说，本文件对应于Linux内核中include/linux/fs.h和include/linux/file.h两个文件

## file.h相关

由于file.h内容相对较少，先说file.h对应的部分

在Linux的file.h中，引用了在fs.h中定义的file结构体，并定义了一堆，基于文件指针fd以及struct fd的操作

对应的在file.rs中，实现了file结构体

```
pub struct File(pub(crate) UnsafeCell<bindings::file>);
```

他反正这样子直接绑定file结构体去了

在File上实现的方法有

- from_fd -> fget

- from_ptr

- pos

- cred

- flags

除此之外还为File实现了AlwaysRefCounted trait，其中包含了inc_ref和dec_ref两个方法。

另外，为fd（不是struct fd，而是一个u32数字）封装了一个FileDescriptorReservation结构体。

实现的方法有（后面的是封装的方法）

- new -> get_unused_fd_flags

- reserved_fd

- commit -> fd_install

以及实现了drop trait

- drop -> put_unused_fd
  
  

## fs.h相关

这里主要实现的是一个file_operations

具体实现方式如下（相当复杂）：

首先从一个宏定义说起：from_kernel_result！宏，该宏的定义本身放在kernel/error.rs中，最后实现为from_kernel_result_helper函数，根据结果来match

随后定义了一个OperationsVtable，在其中实现了和Linux中具体file_operations的绑定：

```
impl<A:OpenAdapter<T::OpenData>,T:Operations> OperationsVtable<A,T>{    //注意这里面的泛型
    unsafe extern "C" fn ...    //file_operations对应接口的具体操作
    {
        from_kernel_result!{
            ...
        }    
    }
    .....
    const VTABLE： bindings::file_operations = bindings::file_operations{
        ...    //分别将file_operations的具体函数和上述的实现相绑定，部分直接没有实现，使用了None
    }
    pub(crate) const unsafe fn build() -> &'static bindings::file_operation{
        //使用这个返回对于file_operations的引用
        &Self::VTABLE
    }
}
```

注意这里面的泛型，包括了OpenAdapter，Operations两个trait。

这两个trait分别在后面实现了

```
pub trait OpenAdapter<T:Sync>
{
    unsafe fn convert(_inode:*mut bindings::inode,_file:*mut bindings::file) -> *const T;
}
```

以及

```
pub trait Operations{
    fn open()
    fn release()
    fn read()
    fn seek()
    fn ioctl()
    fn compat_ioctl()
    fn fsync()
    fn mmap()
    fn poll()
}
```

非常无语的是，上面的这几个函数，除了最后的poll，其他的都是返回ERR，虽然这是默认的操作，但是鉴于上面的OperationsVtable部分也没有实现这几个函数接口，因此，换而言之，这部分的实现按我的理解，就是没有的。

最后，在Operations这里面需要调用若干数据结构，因此，在file.rs里面也进行了实现，主要是两个：polltable和IoctlCommand

分别如下定义

```
pub struct PollTable{
    ptr:*mut bindings::poll_table_struct,
}
```

以及

```
pub struct IoctlCommand{
    cmd:u32,
    arg:usize,
    user_slice:Option<UserSlicePtr>,
}
```

加上一堆在相应数据结构上的操作。

## 其他

除了上述在file和fs头文件中定义的，在file.rs中还有另外一部分内容。

flags，这部分以O开头的flags，并不定义在fs.h，相反，找了一圈，竟然定义在fcntl.h中，是一堆和具体arch相关的宏。我不太清楚这是干嘛的，但是在文件系统各个组件中都有用到，是和具体I/O文件相关的。

但是在整个file.rs文件中并没有被使用。



整个大概就是如此，封装了File结构体以及实现了对应的file_operations。



# kernel/fs.rs

按照注释所说，本文件对应于Linux内核中include/linux/fs.h文件



和上面的file.fs一样，同样搞了一个Tables来管理super_operations

```
struct Table<T: Type + ?Sized>(T);
impl<T:Type + ?Sized> Tables<T>{
    const CONTEXT: bindings::fs_context_operations = bindings::fs_constext_operations...
    unsafe extern "C" fn
    ......
    const SUPER_BLOCK: bindings::super_operations = bindings::super_operations...
}
```

这里面同样使用前面file.rs的方法定义了两个operations。

随后定义了一堆flags

另一部分则是在于file system的注册，实现了一个registration类，用于向系统注册文件系统。包括new，register，unregister_keys以及调用C的init_fs_context_callback和kill_sb_callback以及drop等方法。

还有就是对于超级块的管理，超级块是一个文件系统必须要的部分，分别实现了以下数据结构

```
pub struct SuperParams{
    pub magic: u32,
    pub blocksize_bits: u8,
    pub maxbytes: i64,
    pub tim_gran: u32,
}
pub struct NewSuperBlock<'a,T:Type+?Sized, S=NeedsInit>{
    sb: *mut bindings::super_block,
    _p:PhantomData<(&'a T,S)>,
}
pub struct SuperBlock<T:Type+?Sized>(
    pub(crate) UnsafeCell<bindings::super_block>,
    PhantomData<T>,
)
```

真正的superblock是后面那个SuperBlock，而之前的NewSuperBlock仅仅是在后面实现了一个init_root方法，返回了一个SuperBlock的结果。这就是一种面向对象编程的思想，初始化某个对象的方法，也要作为一个对象来管理？？？

这个newsuperblock也很有意思，在他后面的S属性中，一开始是NeedsInit，经过init之后，该状态转为了NeedsRoot。于是可以调用状态为NeedsRoot的init_root

```
pub struct NewSuperBlock<'a,T:Type+?Sized, S=NeedsInit>{    //S是NeedsInit
    sb: *mut bindings::super_block,
    _p:PhantomData<(&'a T,S)>,
}    //一开始状态，需要调用init
impl<'a,T:Type+?Sized> NewSuperBlock<'a,T,NeedsInit>{
    unsafe fn new(sb:*mut bindings::super_block) -> Self{    //新建一个
        Self{
            sb,
            _p:PhantomData
        }
    }
    pub fn init(self,data:T::Data,params:&SuperParams)
        ->Result<NewSuperBlock<'a,T,NeedsRoot>>        //注意这里面的结果，S转成了NeedsRoot
}
impl<'a,T:Type+?Sized> NewSuperBlock<'a,T,NeedsRoot>
{
    pub fn init_root(self) -> Result<&'a SuperBlock<T>>    //再调用这个的init_root，于是就完成了
}
```

显然他的方法是先new一个，然后状态转为needsinit，再调用init，转为needsroot，最后调用init_root才完成。



除此之外，还分别实现了inode和dentry的部分——但是仅仅调用了bindings::inode和bindings::dentry

还有就是Filename。该部分同样仅仅只有一个init。



最后，定义了一个module_fs的宏，来实现fs模块。

该宏调用了module宏，最后实现了fs模块。（这个模块是指的Linux内的模块，而不是rust的模块）



# kernel/fs/

在这个文件夹下面，仅仅只有一个param.rs。

按照对应注释所说，本文件对应于Linux内核中的include/linux/fs_parser.h，根据描述，这个文件作用在于提供filesystem的参数描述和parser

同时，他也引用了上面的file.rs和fs.rs

在fs_parser.h开头，定义了一系列函数：

```
typedef int fs_param_type(struct p_log *,
            const struct fs_parameter_spec *,
            struct fs_parameter *,
            struct fs_parse_result *
            );
fs_param_type fs_param_is_bool,fs_param_is_u32,fs_param_is_s32,fs_param_is_u64,
            fs_param_is_enum,fs_param_is_string,fs_param_is_blob,fs_param_is_blockdev,
            fs_param_is_path,fs_param_is_fd;
```

而在param.rs里面，他做了对应的事情，但是也没完全实现（可能是由于语言的问题）

定义了一个宏：define_param_type！，该宏中，定义了一个mod，以及spec和handler两个方法。

然后使用该宏定义了一堆mod，bindings绑定了这些函数中的若干个（并非全部）

以上是前250行做的事情。

在后面的代码中，主要是对fs_parameter_spec结构体的管理

其定义基本和Linux中的结构体一致

```
const ZERO_SPCE:bindings::fs_parameter_spec = bindings::fs_parameter_spec{
    name: core::ptr::null(),
    type_:None,
    opt: 0,
    flags: 0,
    data: core::ptr::null(),
};
```

有所区别的是对于这个结构体的管理上面

在rust这里面使用了SpecArray和SpecTable等等一串结构体来对fs_parameter_spec来进行封装管理。

最后定义了一个宏define_fs_params

还记得之前说开头定义的define_param_type这个宏，定义了一堆mod嘛

在define_fs_params这个宏里面，根据传入的参数，对各种各样的类型的mod都调用了一遍对应mod中的spec和handler。



相关操作可以参考一个例子，在samples/rust/rust_fs.rs中有实现一个RustFs最后调用了这个define_fs_params宏，去定义fs的参数。

而最后这个RustFs，又作为一个参数传递给fs.rs中的module_fs宏，定义一个fs

# 与linux内核对比

首先，读代码下来的感觉是，这就是对文件接口的一个封装，而具体屏蔽下层不同文件系统的差别的作用，本身是由Linux实现的，只是用bindings进行了一个调用。

## file.rs

这里面对于File结构体的操作，无论是按照file.h里面给出的文件操作接口，还是按照file.rs中，对于File结构体的读写情况——在这里，他实现了读出f_pos，f_cred，f_flags等等结构体成员的，但是只实现了这些，其他的结构体成员就没管。

而FileDescriptorReservation里面，也只是封装了部分基于fd的操作。

因此，如果需要绕开对于file的operations，主动操作file结构体，那么仍然有大量对于file结构体的操作需要实现



对于file_operations的设计，我个人觉得其实有点无语，按道理，在后面定义了的那个Operations应该是做好了良好设计的一套接口（相对而言）但是我不太明白

## fs.rs

对于fs.h

其实在file.rs里面实现了operations，但是具体绑定的实际操作函数，仍然有大量的函数直接写了None，也就是没有实现。

对比fs.h，主要的file_operations和superblock的operations，都实现了，但是没有实现inode的operations。

总体而言，啊，按照他给出的操作接口操作就完事了。有啥自己想要弄的操作，实现一下就好了。
