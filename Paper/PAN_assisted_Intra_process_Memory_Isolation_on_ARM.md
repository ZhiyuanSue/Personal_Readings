# Introduction

大概就是说，本来是通过进程隔离，现在是用PANIC这个东西，在内核模式下执行进程。并且分为可信任域，和不可信任域

他提到，目前ARM中有Pointer Authen- tication 和 Memory Tagging Extension这些，但是Aarch64还没有高效的（没理解）

目前有

1/基于域的隔离

2/基于地址空间的隔离

基于地址的隔离，我认为就是普通的进程地址空间的隔离，Software Fault Isolation (SFI)，这种技术是基于地址的，检测所有不信任代码的内存访问行为，但是这一定会导致性能问题。

基于域的隔离（看的迷迷糊糊的），就是访问目标内存之前开启权限，访问之后，就撤销权限。例如memory domains 功能，但是这个功能在aarch64上面没有启用，从而提供进程内部的内存隔离。

现在的问题是，aarch64里面，没有提供有效的进程内的内存隔离技术，所以作者提出了PANIC

PANIC依赖于aarch64的Privileged Access Never (PAN) feature 和 load/store unprivileged (LSU) instructions，PAN技术是禁止内核访问用户代码，因为这样有一定概率导致，return-to-user attacks，我看翻译的意思应该是回调函数的攻击。

如果确实需要访问，一个是禁用PAN，另一个是使用LSU指令

在PANIC，他使用内核模式进行访问。然后将需要访问的受限制的内存放在用户模式。（确切的说，感觉就是页面的用户模式位置位。）



但是有个问题，在内核模式下访问会带来更多的威胁

所以他做了两个事情

1、shim-based memory isolation that offer memory isolation between the kernel and the protected process

分为 user shim和kernel shim，通过这个垫片代码去进行访问，使用ASID来隔离TLB条目

2、sensitive instruction emulation that emulates these instructions on the fly.

识别了874条敏感指令，分为无条件敏感指令和条件敏感指令

在执行用户的页之前进行二进制检查（听上去很厉害的样子）

然后将这些指令转换成引发异常的指令。然后在异常中，就让他像普通的用户指令一样去执行



作者在Apple M1上面进行了运行



