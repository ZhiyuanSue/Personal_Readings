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

定义了一个宏：define_param_type！

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
