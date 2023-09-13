# RSDP

root system description pointer

他包含了两个指针，一个XSDT指针指向XSDT，还有一个是RSDT



OSPM必须从平台获取RSDP，然后去定位RSDT或者XSDT



在IA-PC上

扩展BIOS数据区的前1kb，EISA或者MCA系统，在40：0Eh中的双字区域找到。

0E0000h到0FFFFFh之间的Bios只读空间



UEFI查找RSDP

EFI系统表中存在指向RSDP结构的指针。

存在两个ACPI1.0和2.0的指针。定义了两个GUID由这个东西去找。先找当前版本的GUID的RSDP结构体指针，找不到就使用1.0版本的那个



RSDP结构

见表5.3，抄就完事了。

其中还涉及到了ACPI1.0和2.0，扩展结构体的类继承罢了，小意思小意思。



# RDTS

头部加上一堆entry，entry则指向其他表

# XDTS

头部加上一堆entry，entry则指向其他表

# FADT



# GAS

用GAS在ACPI定义的表中描述寄存器地址的结构

（一句话，用来描述寄存器的）

GAS地址空间ID又有好几种格式

这个是这样的我看了，在给定一个地址空间的情况下，那8bytes的字节的地址的说明

地址空间为0和1，分别是系统内存和系统I/O，32位的高DWORD都必须为0

PCI配置空间



PCI bar target，用于在PCI设备BAR空间上定位MMIO寄存器，

地址字段格式分别是8位PCI段，8位PCI总线，5位PCI设备，3位PCI功能，3位BAR索引，37位偏移量。

再回头看PCI配置空间，他必须PCI段为0，PCI总线位0（后面的没看懂



# System Description Table Head

所有的系统描述符都有的一个头部，除了RSDP和FACS



# SCI中断

System Control Interrupt

SCI全称是System Control Interrupt（系统控制中断）当硬件产生了一些ACPI事件的时候,它可以使用系统控制中断SCI来通知OS。SCI是一个低电平有效共享的中断信号。ACPI Spec说有2种类型的event会产生SCI，一种叫做Fixed-FeatureEvents，另外一种是General-PurposeEvents。Fixed-Feature Events 产生的SCI通常是由OS inbox driver去处理的。所以我们只需要关注GPE 。

SCI中断的相关信息被记录在FADT table中，OS通过FADT获得SCI使用的中断号码，从而能够在SCI产生时处理该中断。

通常情况下SCI_INT会被设置成IRQ9，但是它也不是固定的，我们也可以按照板子的设计修改SCI_INT。
