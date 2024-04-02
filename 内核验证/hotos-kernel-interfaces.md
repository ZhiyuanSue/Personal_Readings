## 背景

隔离和单一所有权

## 硬件厂商做的努力

x86最开始用了分段和分页，但是x64模式弃用了分段，只有分页机制

从Skylake开始有了MPK

可以看看这篇阅读

[[论文分享\] Intra-Unikernel Isolation with Intel Memory Protection Keys (mstmoonshine.github.io)](https://mstmoonshine.github.io/p/intra-unikernel-mpk/)

大概思路是给unikernel做一些防御，对于unsafe的代码，每次读写，都改他的MPK权限，然后在执行完之后再改回来

能够进行一定的防御。

但是本身在ring3就可以做这个事情让他具有的安全性更低



对于ARM架构

有MTE memory tag extension，需要和SFI技术配合。

[ARM-指针认证 - 简书 (jianshu.com)](https://www.jianshu.com/p/62bf046b7701)

我的感觉，通过一些东西生成一个密钥，然后这个密钥用来验证。

## 跨隔离边界的攻击

用于隔离的方法有，硬件隔离的原语，软件故障隔离，编程语言安全

硬件隔离的问题是，试图单独维护一个硬件相关的隔离的副本，然后这两个副本需要同步，他对共享的数据结构，是具有写权限的时候，会通过更改共享数据，来影响内核

software fault isolation，使用单个数据副本，但是会有高一致性检查的开销

