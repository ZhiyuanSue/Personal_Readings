对于微内核最小必要部分的思考：

1、 页表管理的部分（但不应当涉及内存管理策略，我不确定这件事情，因为像sel4这样启动之后把剩余内存都扔给进程来管理，是困难的，可以考虑像Redleaf那样，有个buddy用于统一管理模块的划分内存，每个模块内部自己整一个内存分配算法也是可以的吧，redox文档里面没细讲内存管理，估计也是来不及看了）
2、 内核栈管理（sel4内核只有一个栈，但是也可以多个线程各自实现一个栈。）
3、 线程和TCB相关（这部分需要额外设计，比如zicron有job process thread三层划分，job和权限管理是相关联起来的，但是，sel4只有thread，arceos如果算上Starry是有task的）
4、 context  switch和调度策略的接口（但不应该包括调度策略，不过我认为，可以提供简单的一个rr，如果存在用户定义的，就用用户定义的）
5、 IPC方法（比如sel4的端点，redox使用了URL schema，zicron是一个内核对象，然后通过channel和socket进行通信。我个人的看法是建议使用socket，因为这可以搞什么多端融合，被网络模块复用，将远程资源和本地其他服务视作相同的资源。）
6、 类似于capability的权限控制方法。
7、 trap和中断处理相关模块
8、 对于内核对象的锁和同步异步处理。
9、 boot启动部分





其他

也可以看看QNX

[myqnx.com/developers/docs/6.3.0SP3/momentics/bookset.html](http://myqnx.com/developers/docs/6.3.0SP3/momentics/bookset.html)

但是说实在的QNX的调用接口比起sel4多了不少，离最小的原则有一定差距。

