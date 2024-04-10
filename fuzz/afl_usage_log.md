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

LibAFL的一个bear-bones案例


这个案例给了这么两个： executor和 state
使用CommandExecutor::builder和StdState::new生成

state对象 tracks the set of input test cases
任何solution test cases
以及其他元数据

