# 写在前面

啊，还是得去认认真真看代码。

我是长不大的小宝宝~~

还有我必须说明，这只是框架！这只是框架！！！（具体的实现细节我不去讲的！！！）



# 网络部分框架结构



- 用户APP层（雨我无瓜，什么鬼，什么叫socket都没用过。。。）

- socket层（协议无关接口层）

- 协议层

- 网络设备无关接口层

- 网络设备驱动层

- 网卡（驱动顺便讲讲网卡的接口，但是网卡内部实现也与我无瓜）
  
  

其他

- sk_buff 

- net

- ...(东西太多不想讲)

# sk_buff及相关操作

sk_buff起到的作用是作为从上到下或者从下到上传递的包（解释一下sk_buff干嘛用的）

在include/linux/skbuff.h头文件中被描述

具体实现在net/core/skbuff.h



先看数据结构，sk_buff_head用了个双向链表加上spinlock用于锁

sk_buff同样加了双向链表

有一个网络设备字段（收包是可以确定的，但是发包是何时要用哪个网络设备发的？这是比较复杂的，因为有虚拟网络设备到具体网络设备的转变，所以有些情况下会发生改变）

据说古早有什么input_dev设备，现在就一个了。

sk字段指向具体的sock结构体。



几个长度字段：

len，总的长度

datalen，数据的长度

mac_len，mac头的长度

truesize，整个sk_buff管理的空间长度。



几个指针字段

head，end分别是缓冲区的头尾

data和tail分别是数据的头尾



分片操作

在协议中，数据包超过长度之后，分片实在是太正常不过了

skb_shared_info结构体，他是从end指针开始的，所以，还是要留一个end指针的。

里面有两个成员

struct sk_buff *frag_list;

skb_frag_t frags[MAX_SKB_FRAGS];

然后用这个链表的方式去传递给下一个分片。

使用skb_shinfo宏来获取这个skb_shared_info结构体。



csum，校验和

具体字段实在是太多了，而且有注释，自己需要的时候去看。



对应的操作

操作也好多，只能挑重点了。



sk_buff创建

alloc_skb系列函数

以及netdev_alloc_skb（这个我们之前写驱动也是用到的，只不过是netdev_alloc_skb_ip_align，但他最后还是调用了netdev_alloc_skb）主要是用于网络设备收包的时候分配skb结构体的

（何时创建？）总结：收包的时候是网卡驱动层面创建的，发包的时候，因为各种协议的包不同，所以不一定是在具体哪一层发的。

还有一个fclone的，这个具体去看skbuff.c中alloc_skb的实现。

内核中有两个cache：skbuff_fclone_cache : skbuff_head_cache

反正就是直接从已有的缓冲区中创建一个，所以还加上了引用计数。

```
struct sk_buff_fclones {
    struct sk_buff    skb1;
    struct sk_buff    skb2;
    refcount_t    fclone_ref;
};
```

（为什么这么设计？缓冲区性能问题，因为可能指向同一个数据块）

（但我觉得这个clone并不重要）



队列操作，带queue的一堆操作



添加头尾结构体操作

pskb_expand_head，该函数nhead和ntail两个参数分别表示头尾需要的空间。

skb_pad，尾部填充0，调用了pskb_expand_head。

skb_expand_head，他本身又调用了pskb_expand_head，这部分函数操作就是给他添加头部空间的。



trim系列函数

简单来说，截断

具体的截断内容是指向的数据区域加上后面的分片。



释放

_kfree_skb



# struct net

net这个结构体，放在net_namespace.h里面。

主要用于管理命名空间的。

在这个net结构体里面有几个list_head



struct hlist_head  *dev_name_head;

struct hlist_head   *dev_index_head;

一个哈希表的头

这玩意用于寻找dev

在下面那个net_device结构体中，有一个

struct list_head   dev_list 成员

用于链接在一个namespace中的设备。

而

struct hlist_node  index_hlist

这个成员则是用于按照index进行查找。



其他好像无关紧要。

总之，了解了namespace是干嘛的，以及在每个ns中，具体的网络设备都需要映射到其中，就可以理解了。

（具体实现太麻烦了，有需要我去了解）

# socket层

socket层又被称作协议无关接口层，大概意思就是，为上层调用，屏蔽了具体的协议。



AF 表示ADDRESS FAMILY 地址族，PF 表示PROTOCOL FAMILY 协议族

但是网上说，这事情很扯淡，说很多情况下只有微小的差别甚至一样。。。算了，反正socket里面肯定得填协议族



socket层主要包括以下几个数据结构：

socket（include/net.h)

sock(include/sock.h)

另外，在sock.h也有一个proto结构体



socket结构体是用于上层传递下来的socket操作（socket操作在用户态会调用socket层的接口）



socket结构体

```
struct socket {
    socket_state        state;

    short            type;

    unsigned long        flags;

    struct file        *file;
    struct sock        *sk;
    const struct proto_ops    *ops;

    struct socket_wq    wq;
};
```

挑重点讲

file是因为万物皆文件的思想，所以socket作为一个文件，同样要实现一些文件操作接口

sk则是指向后面说的那个sock结构体

ops是对应协议的ops，会最终用具体的ops来赋值

wq则是用于等待队列



对应的操作

其实上层socket的操作对应下来得看proto_ops里面的操作，从bind，accept什么的基本都是



再看struct sock结构体

和socket结构体的差别是这样的，socket结构体向上面的应用层提供了统一的接口，而sock结构体则是网络层的统一接口。



struct sock结构体里面，有个收包队列sk_receive_queue，就是收到的包会挂载到这上面去。







# 协议层

协议层东西可太多了。狗看了都头痛。额我也头痛。

只看IPv4吧，协议层对上层的接口主要在放在af_inet.c文件中

比如上面说到的struct proto_ops的所有操作，都被具体的指定到该c文件中的某个函数中去。

这里面定义了几个proto_ops

```
const struct proto_ops inet_stream_ops
const struct proto_ops inet_dgram_ops
static const struct proto_ops inet_sockraw_ops
```



## 收包：

### ip层

收包是从下到上，ip层收到的是net/ipv4/ip_input.c中，所以下层接口实际调用的是ip_input中的ip_rcv函数，他调用了ip_rcv_core函数（大概是做了一大堆检查），完了之后调用ip_rcv_finish

以及ip_rcv_finish_core（分发到具体的协议栈上面去）



这里面其实调用了别的文件夹的函数（用于处理ip数据包的其他逻辑，比如路由选择等等）

但是如果是向自己本机传送，最终会走到ip_local_deliver

（ppt里面有个图，里面有转发等等情况。）

继续向上ip_local_deliver_finish

--> ip_protocol_deliver_rcu

最终调用ipprot->handler



这里有个数据结构

struct net_protocol，定义在include/net/protocol.h中。

就几个字段

随后在net/ipv4/af_inet.c中，将其handler具体绑定到某个协议的处理中去。



从而分别调用

net/ipv4/af_inet.c中的tcp_v4_rcv/udp_rcv/icmp_rcv/igmp_rcv等等具体的操作

### 上层协议（以tcp_v4为例）

这部分主要是最后会挂载到具体的sock结构体中的收包队列上面去。

只说tcp吧。

tcp在net/ipv4/tcp_v4.c中

主函数是tcp_v4_rcv，他做了一大堆逻辑判断之后调用tcp_v4_do_rcv，参数是sk和skb

这个函数获取sock的锁

然后调用了tcp_rcv_established



tcp_rcv_established函数在tcp_input.c当中，调用tcp_data_queue。

里面调用了tcp_queue_rcv

该函数里面有一行代码

```
__skb_queue_tail(&sk->sk_receive_queue, skb);
```

意思就是把经过一大堆tcp协议处理之后的skb挂载到sock的收包队列上面去。

从而传送到上面的socket层。



但是要说清楚tcp接收如何转到socket调用的收包函数，还得从上向下再来

原因是，recv是可以阻塞的。这也是那个等待队列的作用。

就是说，如果收包队列上面啥也没有，那么就没有接下去的函数调用了。



上面的proto_ops里面的收包函数inet_recvmsg函数会对应的调用tcp_recvmsg或者udp_recvmsg。

还是就说tcp吧，这函数在net/ipv4/tcp.c中。

他调用了tcp_recvmsg_locked

在这个函数里面

```
skb_peek_tail(&sk->sk_receive_queue);
```

把之前放到队列尾部的包给找出来。

然后调用

```
skb_queue_walk(&sk->sk_receive_queue, skb)
```

遍历整个sk_receive_queue队列去得到对应的包，然后进行处理。



除此之外还得说明一下，在tcp_input.c中，本来按照从下到上，应该调用socket的内容，但是实际上用了很多其他结构体的操作。

sock结构体

inet_sock结构体（在include/linux/inet_sock.h里面）表示用了网络传输的socket，添加了端口等等东西

inet_connection_sock结构体（在include/linux/inet_connect_sock.h里面）特指面向连接的socket

tcp_sock结构体，特指tcp的

这四个结构体依次继承



而由于继承，所以tcp在这里面很多操作都是用了tcp_sock或者inet_connection_sock结构体封装好的操作，直接找sock的操作还真不一定有。



## 发包

发包是从上到下，而且没有阻塞。

同样的，从刚才那个proto_ops里面开始，有个inet_sendmsg。具体根据实际分发成为tcp_sendmsg和upd_sendmsg

这也放在tcp.c文件中实现，调用了tcp_sendmsg_locked



这函数要么直接调用__tcp_push_pending_frames，要么走tcp_push，再调用__tcp_push_pending_frames

而__tcp_push_pending_frames又调用了tcp_write_xmit

要么tcp_push_one，tcp_write_xmit



反正最后都走到了tcp_output.c中的tcp_write_xmit函数

在这里调用了tcp_transmit_skb，最后走到了ip层的ip_queue_xmit



ok，tcp层就这样结束了



ip层发包是跑到ip_output.c里面的ip_queue_xmit开始的

__ip_queue_xmit -> ip_local_out -> __ip_local_out -> dst_output

这个dst_output在include/net/dst.h中

最后不选ip6，而是选了ip_output

又回到了ip_output.c，

走了ip_finish_output

-> ip_finish_output2

最后到

-> neigh_output -> neigh_hh_output -> dev_queue_xmit



最后调用的是下一层的dev_queue_xmit



总而言之，调用过程就是这样的，具体实现不必太过关心，也没法关心，主要是全都是协议的处理内容（建议直接对照着网络协议去看，乐。。。）



# 设备无关接口层

这一层主要是提供了对下层设备驱动的封装



直接看include/linux/netdev.h即可

里面有个巨大的结构体

net_device以及netdev_ops，当然其他什么ethtool_ops什么的都有。



在net_device中，存在指向其具体设备操作，以及什么ethtool_ops操作的指针。

对于这一层来说，既然目的是屏蔽下层的实现细节，那么对于驱动层来说，也就是实现上述ops就行了。

在一个最简单的驱动里面，只有open和发包是必须实现的



这个分配就没那么重要了，重要的是，注册网络设备

register_netdev

这个函数在net/core/dev.c中实现

在这里，他会调用netdev_ops里面的ndo_init，给他做初始化（所以其实下面注册的结构体成员的ops并不是没啥用的）



netif_carry_on

netif_carry_off

netif_carry_ok

这个主要是用于打开网络设备的端口的

通常来说，网卡驱动或者phy驱动会给某个寄存器写值，写完之后，就可以了

（说说我之前写Linux驱动和arceos驱动的这个函数调用情况）



这些是向下层提供的接口，除此之外，我个人认为向上层提供的一些接口也要说一下

上面一层说到收包是调用了ip_rcv，发包是dev_queue_xmit，而这些东西都被封装在net/core/dev.c中。

换句话说，上面的net_device结构体是向下的，而这个上层看到的接口是在这个dev.c中的。



先说发包，dev_queue_xmit，他很自然的调用了__dev_queue_xmit

该函数的实现在net/core/dev.c中，调用了dev_hard_start_xmit去发送，随后xmit_one，netdev_start_xmit，__netdev_start_xmit

最后调用了ops->ndo_start_xmit

也就是上面提到的向下的接口。



再说收包，收包在驱动层会调用netif_receive_skb。随后netif_receive_skb_internal，__netif_receive_skb，__netif_receive_skb_one_core，__netif_receive_skb_core，

直到最后调用了deliver_skb

在这里，返回的是如下的一个代码

```
return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
```



这里面func被注册为了ip_rcv，也就是回到了上层的接收入口。

具体代码在net/ipv4/af_inet.c中

```
static struct packet_type ip_packet_type __read_mostly = {
    .type = cpu_to_be16(ETH_P_IP),
    .func = ip_rcv,
    .list_func = ip_list_rcv,
};
```

然后就转为调用上层的ip_rcv了。



因此，在这一层做好了一堆封装，承上启下



（完结）

# 设备驱动层

（作为写了个驱动的人，本宝宝还是自己解释吧，这看上去比写记录更简单一点）
