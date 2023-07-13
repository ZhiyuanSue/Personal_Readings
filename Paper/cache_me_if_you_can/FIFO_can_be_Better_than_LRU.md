啊，无论如何，看之前说一句标题党总是没错的，好吧，来看看理由。

# 概述

一些FIFO算法，比如CLOCK和quick demotion会更快

# 简介

cache的回收算法是cache系统的核心组件之一。

包括两个用于评价cache回收算法的好坏的标准：

1、用cache miss率来衡量的效率

2、用每秒吞吐率来衡量的请求数量



过去对于cache替换算法的工作主要在LRU上面，然后大概介绍了一下LRU，就是用一个双端队列，访问到就放到队列头，丢掉的时候就丢尾部，理念是这样的：最近访问到的内容更有可能继续被访问到。

然后很多算法都在基于LRU进行改进

ARC , SLRU , 2Q , MQ ，multi-generational LRU ，都是基于多个LRU队列去做的改进

 LIRS LIRS2，使用了不同的提升cache的提升标准。

LRFU , EE-LRU , LeCaR ，CACHEUS ，则使用了不同的标准来定义什么是最近的。

Talus，是最近的工作，则用于提升scan和loop的性能。

除此之外，还有cache的周转率，大概意思是说，LRU需要一堆额外的空间去存储记录这个链表并且经常性的进行更改，这本身是一个巨大的成本，因而像CLOCK这样的改进算法被发明出来，他们可以获得和LRU相近的性能。

于是乎，作者设计了一个方案，相比之下有16倍的traces和58000倍的requests

与LRU相比之下，FIFO有更少的元数据和计算量，以及flash友好的，更好的可测度性，但是FIFO会造成很大的效率余量，因此作者介绍了这么两个方法

Lazy Promotion （LP）以及Quick Demotion（QD）

对于LP，他做的事情是仅仅在替换的时间段里面才进行promotion，过去的建议都是说，带上LP的FIFO，效率近似与LRU，但是相比之下要差一点，但是本文作者说，他们大规模弄了之后，发现效率更好一些

对于QD，在他们插入之后，快速的删除掉他们。因为让整个object穿过整个队列是消耗很大的事情然后报告说，这个带QD的FIFO性能提升挺大，而 QD-LP-FIFO，则更好。



# 为什么是FIFO以及他需要什么

首先是很多论文都论证了FIFO的优点，FIFO更少的元数据，以及元数据更新工作，LRU需要更改六个指针，FIFO具有吞吐量和可伸缩性的优点，LRU优于FIFO仍然被当做一个常识。

那，为此，作者做了一个抽象的cache，一个cache可以看做一个逻辑上按总顺序排列的队列，然后他存在四个操作，insertion , removal , promotion , and demotion ，cache中的对象，可以基于一定的标准（例如，从上一次访问到现在经过的时间），进行比较和排序，以及基于这个标准，删除掉最不重要的对象。

用户可以控制的是插入和删除操作，删除可以直接删掉，也可以通过TTL之类的删除掉，而promotion和demotion则是用于控制队列中的顺序的操作。

多数的算法使用promotion来更新他的顺序，比如LRU将他访问到的那个object放到最前面，称之为eager promotion，而demotion则是隐含的操作——一个对象被promotion的时候，其他所有对象都被被动的demotion了，称之为passive demotion。这是一个缓慢的过程，因此某个对象在被逐出之前，需要遍历整个队列。而作者试图证明，我们应该用 Lazy Promotion and Quick Demotion。



# Lazy Promotion

给FIFO添加上Lazy Promotion，就是所谓的LP-FIFO，只有在将要被替换掉的时候，才进行Promotion，目的是使得使用最小的代价把需要保留的object保存下来。举个例子就是FIFO-Insertion，如果在缓存中，当在替换时间的时候如果被访问到，那么重新插入。

LP-FIFO继承了FIFO的一些优势，首先是，减少元数据的需求，甚至可以只要一个boolean位，其次，在被替换的时候才进行promotion，那么意味着，可以得到更多的信息，比如说在这段时间被访问的次数。

作者拿FIFO-Reinsertion（也就是单bit的CLOCK）和2-bitCLOCK以及LRU进行对比，反正意思就是LP-FIFO的miss rate更小。

即使是在两个social的network数据集中，LRU比单bit的CLOCK更好，但是这是因为他单bit不够用了

更好的原因有两个，LP通常就会导致QD，以及，很多时候，本身数据就有FIFO的要求

但是为了性能更好一点，作者又弄了个QD

# Quick Demotion

LP的目标是让popular的object留在cache中，而QD的目标是让unpopular的object被逐出

每个cache中的object的目标是，让每个object有更多的机会来证明他有价值被留在cache中，而其中大部分的object是unpopular的。但是cache的工作负载通常符合Zipf分布。

这在scanf或者loop中，会更加典型，而其中存在大量的short-live的对象。

换句话说，给这么些新对象这么久的时间，来证明他们的价值，这件事情本身的代价太过高昂，而在队列末尾的被逐出去的那个甚至有可能有更多的概率被访问到。

这想法并不新颖，但是，作者认为他们扔出去的不够快，然后，作者用一个公式来计算各种算法的object消费时间

通常来说，越是有效的算法，在不常用的对象上面消耗的资源越少。

然后作者实现了一个QD算法。用一个小的FIFO来存储cachedata ，然后一个影子FIFO，从那个小的FIFO里面被排除出来的元数据和对象存放在这个FIFO队列中。

那个小的FIFO就当做一个过滤器，只占用10%的cache空间，筛选出不常用的对象。在插入之后没有被访问到的对象就直接从这个队列里面排除出去。

另一个大的，则，占用了90%的cache空间。

在cache miss之前，对象会写入小的FIFO队列，而不是大的FIFO队列，除非已经在大队列中去了。

当小的队列已经满了的时候，如果在此之前已经被访问过，那么就扔到主cache中，否则放到GhostCache中



接下来是性能展示，在web workload中，比起block workload，更好，因为web会有很多unpopular的对象。

但是，Cache太小的时候，加QD会导致，额，没啥用，因为无法捕捉到可能的popular对象。

所以加上LP就是LP-QD-FIFO

剩下的废话不看。

# 讨论

其实还有其他算法来让popular的对象留下来，虽然本文没用到，但是仍然有用的。

QD也一样，甚至有些非常激进，比如说拒绝某个对象进入Cache

其他，没了，这篇论文看完了。


