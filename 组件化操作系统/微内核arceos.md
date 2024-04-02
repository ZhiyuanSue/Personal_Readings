# Redox

arceos的微内核部分是参考了Redox的，所以很多可以参考Redox的框架



Scheme

分为user和kernel的scheme

把所有的内核调用都当成一个文件读写行为。因此，对一大堆的操作都封装成了文件访问。

然后一个用户态的内核模块其实就是通过Scheme来访问的。

一个模块需要向内核注册相应的scheme

Root Scheme，作为管理scheme的一个文件，其下有一堆user的scheme



URL



Namespace对Scheme的集合管理