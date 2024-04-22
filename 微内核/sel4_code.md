# introduction
反正大概就是读内核代码的记录吧。
我试图编译并运行了sel4，具体得看
https://docs.sel4.systems/GettingStarted
这个网址的启动说明
但是我使用的sel4的仓库则是git@github.com:seL4/seL4.git
不过我想问题不大。
这个仓库缺少足够的测试套件，而是纯粹的代码。
然后在编译的时候，提示python缺少足够的包，安装就行
就是那个奇怪的past包需要安装的其实是future
！ 这是什么奇怪的命名方式。
还是从x86的64位代码开始读吧

# head.s
开头用了一大堆的begin func
不管他

总而言之，这里面实现了bsp和ap的启动

然后对于bsp，走到boot sys
对于ap，走到boot node

# boot_sys.c
走到这里，只是检查了一下multiboot头。
然后调用try boot sys
之后再调用schedule
Activetethread

try boot sys()
这里同样，用了一个比较大的结构体，表示boot时候得到的一堆信息。

啊这，他这里检查了一波cpu是不是AMD和Intel的，emmm现在我倒是想买到非intel或者非AMD的x86...
国产x86么？？？

pic_remap_irqs
他说把传统的IRQ重新映射到正确的向量上。
我的理解是，在bios这里面已经做了一些初始化，但是并不是正确的，因为现在我们换了位置了。
但是问题不大因为大概率会用apic然后禁用掉。

如果我们使用apic，需要disable所有的向量

这里面还需要检查ACPI表

然后从ACPI表里面查找可用的CPU

啊，总归就是一大堆相关的配置。

然后试图进行多核启动。

先是走到了try boot sys node

然后ioapic init

初始化BKL

然后start boot aps
这里有个小技巧（我觉的很有意思），简单来说，根据是否启用smp，定义一大堆的宏
然后如果有smp的宏就正常定义，然后没有smp的就弄成空的
这样就可以不用到处ifndef什么的乱飞了。
嗯，好东西

BKL
大内核锁
最大的优点是，实现简单，不用考虑各种奇奇怪怪的同步问题。嗯，真棒
他使用了一种CLH锁
啊好吧至于什么是CLH锁，请自行了解。


start boot aps

反正通过acpi的数组，一个个初始化其他cpu
最后会调用start_cpu
在start cpu中，做了这么几个事情
一个是弄了一个x86的mfence用于同步，相当于memory barrier
第二个是发送初始化ipi，
大概是通过wrmsr写APIC相关的ICR寄存器，（interrupt command register）
反正又分为xapic和x2apic两种。

并且需要不断地等待所有的ap boot up

好，来说AP的启动

## ap start
虽然我也不知道他是怎么指定启动的ap的，但是我觉得问题不大
然后啪叽啪叽走完了汇编，就走到了boot_node

然后是init cpu
这部分将改变CPU的状态。我的感觉只是对他做一个同步。

最后更新最新的一个smp的数量，就行了。

然后这样子，smp的实现就结束了

在ap走到schedule和activateThread之前。

还有一些工作需要完成。

然后这里就开始使用tcb的概念了。

在init core state中，需要加入初试的线程到debug queue中去

额，这复杂的宏，让我实在是一言难尽，
但是总而言之，大概就是用tcb_t声明了一堆东西吧。。。
一个让我觉得那个文档有些问题的地方在于，他觉得tcb只是一个简单的内存空间？？？
无论从字面意思还是我看到的代码，这应该都是表示线程控制块的啊
emmm。。。

然后如果没有smp需要声明的就是idle thread和schedule 的thread的tcb

然后就来到了伟大的schedule中了

在这一步，可以认为bsp和所有的ap都完成了工作

然后就可以正常schedule和activateThread了。

对于schedule，感觉更多是像是选择下一个thread

然后activate才像是真正的激活thread

# init kernel
在arm和rv的部分里面，有这两个函数，跟x86的实现有一些，额好像是巨大的差别。

# 关于内存管理
我今天看了一圈文档





# 关于capability space
我试图直接看代码，但是挺晕的，所以我找了一份文档
https://docs.sel4.systems/Tutorials/capabilities.html

There are three kinds of capabilities in seL4:
- capabilities that control access to kernel objects such as thread control blocks,
- capabilities that control access to abstract resources such as IRQControl, and
- untyped capabilities, that are responsible for memory ranges and allocation from those (see also the Untyped tutorial).

A CNode (capability-node) is an object full of capabilities: you can think of a CNode as an array of capabilities
然后代码不要看sel4那个，要看sel4test里面有足够的代码。
我也看了rel4test的仓库，他貌似是从sel4的这个延伸出来的然后新建了rel4的代码，但是测试相关用的还是sel4的代码

对于cap的定义
struct cap里面包含两个u64int_t,
一个capability space可能包含多个cnode

还是来看代码吧，他给的例子是一个函数seL4_CNode_Copy，这个函数可以去找，找到他的原型
seL4_CNode是一个seL4_CPtr类型，而seL4_CPtr是一个seL4_Word类型。



# 关于IPC

