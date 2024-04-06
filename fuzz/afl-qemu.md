# introduction
由于qemu很受欢迎，所以fuzz很多去选择qemu
几个问题，一个是各种开发的分支太多了，导致很乱，另一个是基于Python等语言开发的，导致他性能不是很好
问题就在于需要一个最新的框架，不用给qemu打补丁，并且需要和最新的fuzz框架比较容易地集成
因此本文中提出了一个lib afl qemu

（作为既不了解qemu的实现细节，也不了解afl的实现细节的人，我只能说，从逻辑上来说，上面的话我是可以理解的）

他说将qemu作为一个库来处理。修改核心的qemu的大约2k行代码
直接控制QEMU的API，就不用将C代码编进QEMU中进行插装

（我开始疑惑一个事情，他是用的qemu system还是只是user的qemu呢）
他说，像kAFL一样（需要看一下），不仅可以执行大部分的x86二进制（按照这个说法，应该是user的qemu）
而且可以执行嵌入式应用程序或者完整虚拟机（也许可以支持内核吧，因为他给了个windows的ntfs内核驱动程序，但也仅仅是一个驱动）

# background
大概就是介绍了
1、fuzz
（不说了）
2、虚拟机相关仿真软件
静态仿真，也包括isa和宿主机的isa相同的情况下的优化，还有不同的时候的硬件直接模拟（讲真直接跑就直接跑吧，调试一个内核也没有特别慢感觉）
动态二进制转译
（DBT，Dynamic  Binary  Translation)
静态二进制覆写
（SBT，Static  Binary  Translation）
将二进制重写，完全重写为另一个架构的东西，一种广泛的方式，是重写为现代编译器的中间表示，所以很多fuzzer也经常用。
3、两种模糊测试方法的缺点
SBT其实还是很有前途的，因为他是静态改写的，所以效率会高很多

但是DBT也是有他的优势，毕竟SBT只关注与少数几个ISA
巴拉巴拉，反正比较一下两个方式
3、QEMU
当然可以依赖于KVM做硬件辅助虚拟化，但是他首先翻译成TCG（Tiny Code Generator）的中间表示，经过优化之后，翻译成主机的一个机器语言
然后他的一个编译单元是一个TB（翻译块），相当于一个基本块
他会具有多层的TB的缓存，还可以动态的将TB链接在一起加速
还会允许执行一些C代码，比如用于改变机器的状态，称作辅助函数。
还包括各种显示等各种输入输出驱动

# Design
## general overview
首先是将qemu作为一个库，并且将他的工作原理暴露给rust
首先确定使用的qemu版本
用的qemu-libafl-bridge版本分支
然后做了两个库，加上qemu本身的库，静态通过FFI给rust的crate
1、libafl_qemu_build用于提供一个API来构建QEMU，动态使用bindgen生成bindings
2、libafl_qemu_sys将C的FFI绑定暴露给Rust的库
3、主要的库是libafl_qemu
需要正常的在fuzzer crate中调用qemu，只用cargo build即可

划分为三个层次的抽象
1、低级API，exposed by the Emulator singleton（没懂）
接近于libafl_qemu_sys并且用safe rust 封装
2、高级API，基于低级API的抽象，提供hooking功能，可以像正常的代码一样与LIB AFL fuzzer其他状态和hook交互，实现address sanitizer等功能（这是个啥？）
3、基于低级和高级API的面向模糊测试的构建的功能，大概就是跟afl等fuzz库的一个交互的接口吧
基本功能上面，是支持qemu所有的架构的
但是额外的fuzz功能，只支持少数几个架构。
