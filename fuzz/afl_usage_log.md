# LibAFL introductory WorkShop

一些资源的介绍
libafl的maintainer写的
https://aflplus.plus/libafl-book/

fuzzing 101
https://epi052.gitlab.io/notes-to-self/blog/2021-11-01-fuzzing-101-with-libafl/

LIBAFL本身具有的example
https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers

Fuzzers

反正大的意思是一个loop，不断地根据种子生成变异输出样例，然后win就是引起了一个core dump

## LibAFL的一个bear-bones案例


这个案例给了这么两个： executor和 state
使用CommandExecutor::builder和StdState::new生成

state对象 tracks the set of input test cases
任何solution test cases
以及其他元数据

然后 CommandExecutor会执行我们的目标程序，然后传入我们的测例

然后加上mutator，会设定maximum mutation iteration
最大迭代次数

加上stage，设置了一个简单的pipeline，只有一个stage
这个会从一堆随机的mutation中为每个test case选择

然后设置了scheduler和fuzzer
使用state.load_initial_inputs_forced来设置随机corpus

最后fuzzer.fuzz_loop就跑

设置了feedback，设置为ConstFeedback：：False就是期望一个CrashFeedback

### 使用custom feedback

由于没有有趣的反馈，那么很难再迭代出来有趣的输入

那么如何获取这些有趣的反馈呢

使用fuzz_target target_dbg，可以从stderr得到一些输出信息

如果我们之前没有见过这些debug信息，那么就认为是有趣的，值得迭代的

而这个在libafl里面没有，需要自己实现

StdErrObserver结构，可以让我们使用自己的CommandExecutor

我们需要自己生成一个is_interesting方法，为Feedback 这个trait

提供的入参为state，mutated input，observers

我们的期望是，这个事情吧，他可以尽快的弄到一个可行的解导致报bug的

在这个案例里面仅仅简单的seen hashes

但是这种情况，会很容易的就导致他崩溃。。。

### code 覆盖率的feedback

如果只使用简单的stderr的output，是不是那么可靠的。

可以使用code coverage这个指标作为feedback

通过观察哪些block被执行，可以看到哪些输出在探索有趣的逻辑

可以插装

AFL++使用一个改过的clang的版本

使用shared memory

使用ForkserverExecutor（看名字应该是使用fork的方式做的，可以用shared memory，这里有个shmem_provider参数，填入共享内存

使用HitcountsMapObserver使用共享内存

使用fork的一个好处是说，可以加快速度，

使用这种方法，他更快地找到了fuzz的路径

### Custom Mutation
目前为止我们使用havoc_mutations

当然也有一些奇奇怪怪的mutation，可以在AFL++那里找到

然后他说，在除了大小写字母和数字之外的部分的字符输入，都被拒了，因为确实不符合情况嘛，对吧
然后这也很浪费时间

如果我们知道我们的目标，就可以按照需求去改

使用一个mutate函数，去更改变异的范围。

Persistent Fuzzer

这个意思是说，之前的执行器，比如fork server每次都新开一个线程，实在是牛逼的想法，但是还是太慢了，为啥不每次都新开一个线程可以测试好几次呢

被称为persistent mode fuzzing
哈哈哈哈AFL++的手册还说
Basically, if you do not fuzz a target in persistent mode, then you are just doing it for a hobby and not professionally :-).

Harness
额，大概是说，因为他会改动全局的变量什么的，所以会上一个harness（我查出来是什么马具，之类的含义，估计是什么限制吧）

just a bit of code we write to call in to the target for fuzzing
