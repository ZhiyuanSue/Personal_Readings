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

lock-free
可能会帮助其他线程的操作完成，因此必然会在有限操作中完成

wait-free
但是lock free会导致，可能会不断有新的线程去操作，而当前进程需要先帮助他完成，从而会导致永久的活锁被饿死。
通常在实时硬件上，会要求wait free，但是lock free只会在通用的多处理器中考虑

obstruction-freedom
主要的问题在于需要帮助他人执行，这是无锁算法的复杂性的来源

## 理想算法的特征
disjoint-access parallel

linearisability

## 硬件原语
CAS，DCAS
但是DCAS只有某个奇葩的摩托罗拉680x0才支持
后面加了一大堆的列举背景的实现，建议不看，反正不如本文

包括MCAS 和software transactional memory（STM）

### MCAS
是CAS的直接扩展，由一组元组（address，expected，new）指定，
如果所有位置都包含期望的值，那么所有位置都会更新为指定的新值。
LL/SC指令
### STM
事务性内存
利用现有的多处理器缓存一致性的硬件设计。
但是需要改硬件
也有更为困难的设计
反正一大堆，就是说了一堆background，各有各的缺点

## 没有考虑复杂的数据结构

## 内存管理
许多语言其实没有考虑自动垃圾收集器。认为他会被自动的回收掉
但是事情并非总是如此。

# 第三章 实用的无锁编程抽象
应该是开始正文了

所以首先作者先展示了一个MCAS的实现和一个STM

## MCAS

他应该包含两个要求
1、自动执行的，一个成功的操作，必须立即用新的值来替换他旧的值。
2、如果失败，那么必须能够撤销所做的任何更新。

作者的思路是一个两阶段更新法
第一个阶段获取每个位置的所有权，如果所有的都成功了，那么被认为是成功的，
第二个阶段是更新
如果第一个阶段成果，则将更新为新的值，否则将其恢复

他在图3.2中给了一个伪代码。
