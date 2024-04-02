一个有用的链接是



https://gitee.com/openharmony/docs/blob/master/zh-cn/readme/%E9%A9%B1%E5%8A%A8%E5%AD%90%E7%B3%BB%E7%BB%9F.md

（其实相关的说明都在readme的中文版里面有）



首先是下层的os的支持的情况

在这里面支持了linux和liteos等等

## interface/inner_api/osal

这个文件夹下面基本都是对于os的api的抽象层

感觉就很像一个trait

仅仅提供了接口，但是具体的实现则是在adapter里面的（以linux为例）的khdf/linux/osal/下面



就是，对于驱动所必须的os的接口，进行了一堆抽象

然后再adapter里面，对这些驱动必须的抽象做了实现

这是os抽象层做的事情



## framework

framework/include/里面包含了一些比如net，camera等等基本的设备驱动的模型，他通过对这些东西进行建模保证对于不同的设备有不同的操作模型

同样的framework/model下面也有一堆东西，对于不同的设备进行不同的建模。

对于这两个模块下的内容的区别

还是举个网卡的例子，model下的network，涉及到不同的网络设备比如蓝牙，ethernet，wifi等等，那么往往也有不同的实现。



而include中的net的设备，通常则是他的共性的部分。



## 如何将自己抽象的网络模型，跟linux或者其他liteos的网络模型兼容？

我仔细看了，我真的认真看了，觉得有些事情泄漏还是没办法的，他确实设计了这套ops做的还挺可以好像，但是，下面有个代码

```
#if defined(CONFIG_DRIVERS_HDF_IMX8MM_ETHERNET)
......
struct sk_buff* (*alloc_buf)(struct NetDeviceImpl *impl, uint32_t length);
......

#endif
```

整一个大无语，这种sk_buff，是专属于linux的，可以在这里定义的吗

从只是要跑起来的角度，这可能问题不大

但是，从一个抽象层的角度来看，这就不是什么好设计



回到正题，NetDeviceImplOp这个里面，确实传入的参数都尽可能是这个鸿蒙定义的东西，返回一个int

对于linux的比如net_device的数据结构的申请上面，都是尽可能的包含在这些函数的具体实现中，而不让他泄漏出来了。

（上面那个例子，说明设计的不好，有问题）



除此之外，他的model之外还有一个platform（我不理解为什么要分开设计，问了萧络元也不知道），这两部分也是分开设计的。但是总而言之，这些具体实现只是有限的抽象，对于linux内核驱动很多地方，都很难讲能够直接迁移过来




