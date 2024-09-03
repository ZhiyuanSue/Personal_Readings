abstract

摘要说，总结了一下抽象的模式。aliasing and mutable和aliasing xor mutable

他说rust是强制的后者。

但是缺少了前者。



motivation

他指出了两种模式的不同：

一个是lock protected values

一个是process owned

前者好理解，也就是一把锁，后者是说，只有调用对应的syscall的时候，才会需要用到这个value

在xv6中，因为每个进程单线程，所以后者就直接写进pcb中

他举了个例子就是process的cwd，表示process当前使用的路径

使用chdir系统调用去改变



作者于是写了两个版本的代码

由于他并不是多线程的进程

所以实际上用锁的那个版本性能是不好的

也就是说，本来没有人会抢mutex，但是你还是要mutex一下，那就不太妙了



作者说，传统的遗留操作系统中缺少所谓的A&M模式抽象

而Rust的模式，选择更多的依赖于模块而不是使用传统操作系统中的这种A&M模式抽象

（显然，从此得出的结论是，使用rust然后就用了锁保护值来取代进程拥有这些value的模式，但显然前者按照前面的说法，性能并不好，所以，看上去更安全了，但是在这个毫无意义的锁上面花费了太多的资源，导致性能变差，这是使用rust这种语言的模型所带来的问题。）



Background：Alias and Mutable和 Alias Xor Mutable

（讲真，前面我看到这两个名词表示一头雾水）

1、安全rust的 Alias and Mutable模式

ownership

就是某个东西的ownership被传给了函数，后面访问不了了，rust基本操作。

mutable reference

通过可变引用来使用

shared reference

就是共享引用

因为多个共同使用一个，所以保证了不能同时使用它，也就是说，没法同时修改。



2、不安全的rust的A&M

这种情况下限定为Alias Xor Mutable，

也就是放弃安全保证，比如原始指针的解引用。



为了解决不安全rust的这些问题，现在有几种方式，反正就是前面说的patten，比如锁保护，这种方法降低了安全推理的成本

但是比如get_mut_unchecked方法，调用者本身必须确保没有其他线程正在访问相同的数据，但是即便如此，这也能减少安全推理成本，因为，限制在了类型中。



A&M模式下的不安全操作

Rust本身也提供了unsafe操作：raw pointer和interior mutability

裸指针比如说*mut T，额不说了

UnsafeCell< T >, 内部可变性，虽然shared reference是不可变的，但是反正使用它的时候会认为它内部是不可变的去优化。



A&M模式的example

这里给了两个example

1、原始指针实现的自引用值

自引用的含义是，包含指向自身的指针。

为了防止悬空指针问题，它不能移动。

在安全rust中，他是不可以移动的，编译器会阻止他，但是在不安全的情况下，这是不正常的

所以引入了Pin类型。



2、具有内部可变性的锁保护

这种模式是这样的

mutex< T > 里面包含了UnsafeCell< T >和lock

然后Guard则是对Mutex< T >的一个封装，可以自动调用析构函数，防止调用之后不释放锁。

和C++里面lock_guard不同的是，C++仍然可以在不持有锁的情况下访问，从而存在问题。



这两者的比较，就是如何选择

动态生命周期的，比如Rc，应该用原始指针，

而如果是具有静态确定生命周期的A&M则需要使用内部可变性来实现。

因为后者并不会损坏模块化，会把东西限制在他内部



作者说观察Rust的标准库本身可以得到这样的结果。



作者为了发现操作系统中的A&M模式，重写了xv6操作系统

（我严重怀疑，他是先有了重写的结果，然后再去做的这个论文）

他绘制了类型之间的依赖关系图，然后系统的识别这种模式。

然后它说它识别出来六种A&M的模式和抽象



1、Process-owned value模式

他举得这个例子里面，一个不好的做法是，提供一个&Proc作为参数并返回一个&mut OwnedData。

因为这样其实会允许多个可变引用。

所以他封装了一个对进程的共享引用。指向当前进程。

然后只允许有一个current proc。

这时候，只有在user_trap中才会调用，这是内核的入口点。

let proc = unsafe { current_proc() }

一个很显然的事情是，一个core同一时间可能会中断嵌套等等，但是对于系统调用，同一时间只可能有一个。

2、CPU-owned value模式

这里有个问题是，线程在多个core之间的迁移，这里作者提出了一个办法是，禁用抢占或者cpu间的任务迁移

而不是禁用中断



禁用中断他使用了一个tifflin kernel的方法。有个不变值，如果存在push ooff塞进去的heldinterrupt值，他必须被禁用。



3、Memory Pool

反正一句话，这个可以用一个锁去实现。

而作者的看法是使用refmut来实现读写

但是后面的部分，我确实缺少对于rust的了解。感觉心累。

4、Lock-protected immovable value

就是对于不可移动的值的保护，比如某些奇奇怪怪全局唯一的设备之类的写入。



5、Lock-protected separated value

这里举得例子是，对于父进程。他需要遍历所有进程的父进程并和自己比较判断是否存在子进程是僵尸进程。

有个问题是，所有proc的的parent的字段都应该使用一个single的lock去保护它。否则就会出现问题。

所以他开发了一个branded types。

6、Asynchronous ownership transfer for DMA

我对这个模式的理解是，就是他DMA会在不经过CPU的情况下去修改内存中的数据，这会导致原先的所有权的机制失效。

换而言之，这是由于DMA的异步造成的问题。

那么他这个时候的做法是，确定他的不变量，也就是





评价

这个评价读的我很诧异

他说linux用了全部六种抽象，这没问题。

但是rust xv6没有为这全部六种模式做抽象，他只是做了C到rust的转换（？？？）

额，后面在测试的时候，甚至说，没有完全实现usertests套件的程序的运行。所以实际上，他就是，一个没有完全跑通的xv6的rust版本？？？真无语。







