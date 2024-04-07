# Introduction

互斥锁的实现有各种各样的缺点

关键操作由于调度的延迟
性能的需要
死锁等问题

现有的无锁结构并不合适

## 术语
lock-free
通常被定义为，一个进程停滞，不会导致其他进程无限期的停滞

CAS
sompare and swap
包括三个参数
一个内存位置，一个预期读出的值，如果在该位置有预期的这个值，则会写入，并且还会读出，这是原子的。

另一个是
DCAS
double word compare and swap
同时执行两个CAS操作，当且仅当两个位置都包含预期的初始值，才自动更新两个值，

问题是DCAS到（作者写这篇文章的时候）任何现代处理器架构的硬件都不支持DCAS

recursive helping
在lock-free算法中一个通用的用于处理冲突的算法。

如果操作A的进度收到冲突操作B的阻碍
那么A将帮助B完成工作。
通常使用递归的reentering the operation，并且传递B指定的调用参数。B需要提供足够的信息，允许冲突的进程确定调用参数。
在递归调用完成之后，就不会被阻塞，A可以继续操作。

在使用锁协议的时候，通常讨论两个锁：mutual_exclusion lock，multi-reader lock

## 贡献
第一个是实现了两个编程抽象，基于MCAS（multi-word compare and swap），以及software transaction memory（STM）
（反正作者说他的实现更好）

第二个是设计了三个lock-free的search structure
skip lists
BSTs
红黑树

## 伪代码约定
bool TRUE FALSE
word 表示内存中的一个word
CAS函数CAS（word * address，word expected,word new)
一个使用java风格的一个new，但是假定自动垃圾收集的。
tuples，（type1,...,type n）可以在函数之间传递
赋值语句左边变量名用下划线代替，表示抛弃数据，空操作
下面还列举了一些更加清晰的操作符。

# background

