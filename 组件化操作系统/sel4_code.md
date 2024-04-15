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

