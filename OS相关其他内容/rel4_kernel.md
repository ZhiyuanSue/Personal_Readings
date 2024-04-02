# rust版本的sel4阅读

从src boot文件夹开始，这是启动的部分

主要的入口应该是try_init_kernel（前面应该还有点，但是无所谓了）

最开始是log系统和heap

heap用了个static的mutex来实现分配了1M的内存给他。

然后查了一堆地址的上下限，region啥的，不说他

调用rust_map_kernel_window，不管是啥吧，写的乱七八糟，大概就是做了page_table的地址映射

调用init_cpu，设置页表，设置中断向量，还有SMP的中断的使能

set timer

然后这里面只有qemu下有init hart，去设置了PLIC（这也是多核中断情况下必须考虑的事情）

然后调用init_dtb，反正就是返回了个映射吧。

巴拉巴拉处理了一大堆之后，来了一个root_server_init

（这里面很多代码结构都放在interface.rs中）



感觉有点复杂，去网上找了本中文翻译的书看看

[sel4.0.9.pdf · laokz/seL4内核参考手册 - Gitee.com](https://gitee.com/laokz/sel4_reference_manual/blob/master/sel4.0.9.pdf)



