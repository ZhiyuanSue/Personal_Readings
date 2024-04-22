# 结构
结构很简单，总共只包含了15个文件

具体的cap放在cap/文件夹下面
正如另一个文档记录的内容，sel4的cap是两个机器字，还是先从底层的cap文件夹开始看起

# cap/mod.rs
他枚举了几个captag
然后有个plus define bitfield的宏
按照我的理解，这里面有一些奇怪的数字，是对于不同的内核对象的类型，比如tcb啊什么的这些结构，做的
（就比如对一个untyped实行了retype之后，需要给他按照某个类型模板生成内容，然后需要给他一个cap）

在这里实现了一些CapTag的公共的方法。

（我是真的，emmm他这个，他这个都不全。我算是体会到陈老板说的什么，模块化的目的是为了能够组件复用，这，这完全没法复用啊）

update方法，大概就是，给一个新的字段
剩下几个方法看上去都是简单的一些位操作


same object as/arch same object as和same region as函数
前面先验证了两个cap的权限
然后分类型讨论了一下，比如irqhandler，就需要得到具体的handler

# cap/other files

怎么说呢，基本就是实现了一个简单的cap_t接口，主要是针对于不同的cap_t

# cte.rs
按我的理解，cte是capability的一个entry
cspace的基本组成单元
他说由cap_t和mdb_mode组成

derive cap
主打还是一个根据传入的cap_t来导出派生的一个
