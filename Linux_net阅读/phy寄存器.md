phy这个东西本身是在IEEE802.3中得到定义了的，所以没啥多说的

# MDIO

MDIO使用简单的双线串行接口，用于链接PHY并控制PHY，称为MII Management Interface

管理接口通过一对MII或者GMII传输的信号，帧格式和协议规范，以及通过这些帧读取和写入的寄存器集合组成。

MII使用两个基本寄存器，GMII使用了第三个。

MII基本寄存器  由两个寄存器组成，分别是控制寄存器（寄存器0）和状态寄存器（寄存器1）

GMII在这俩之外还包括了寄存器15（扩展状态寄存器）

这个status和ctrl功能被认为是100Mbps和1000Mbps的基本功能。（也就是说，这三个寄存器才是最基本的）

寄存器2-14是扩展寄存器集合的一部分

寄存器4-10的格式是为了使用特定的自协商协议定义的。这些寄存器格式由寄存器1和15定义

# 寄存器

具体而言，前16个寄存器0-15是有规定好功能定义的

后面16-31是厂家自己设计的。

（但是为什么在phy.h中对后面的寄存器也有呢？？？）

关于这16个寄存器，功能分别如下记录

使用X.Y方式记录，比如0.15就表示第0号寄存器的第15个bit。

这些寄存器都是16位的。

# 0号寄存器-Ctrl（MII和GMII都是基本寄存器）

| Bit(s）    | Name            | Description                                                                                                                          | 读写情况    |
| --------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------- |
| 0.15      | Reset           | 默认是0，位置位1表示reset，该位置为0表示正常工作，每次phy复位应当在0.5s内完成，复位过程中保持bit15为1，复位之后自动清零。需要注意的是，一般要改变端口工作模式，比如速率或者协商信息的，需要设置完相应寄存器之后，再reset phy来让配置生效。 | 可读写，自清理 |
| 0.14      | Loopback        | 逻辑上会把发出去的再送回来，但有些会导致直接linkdown                                                                                                       |         |
| 0.13和0.16 | Speed Selection |                                                                                                                                      |         |



# 1号寄存器-Status（MII和GMII都是基本寄存器）



# 2号寄存器-PHY_id



# 3号寄存器-PHY_id

# 4号寄存器-Auto-Negotiation-Advertisement

# 5号寄存器-Auto-Negotiation-Link-Partner-Base-Page-Ability

# 6号寄存器-Auto-Negotiation-Expansion

# 7号寄存器-Auto-Negotiation-Next-Page-Transmit



# 8号寄存器-Auto-Negotiation-Link-Partner-Received-Next-Page

# 9号寄存器-Master-Slave-Control-Register

# 10号寄存器-Master-Slave-Status-Register

# 11号寄存器-PSE-Ctrl-Register

# 12号寄存器-PSE-Status-Register

# 13号寄存器-MMD-Access-Ctrl-Register

# 14号寄存器-MMD-Access-Address-Data-Register

# 15号寄存器-Extended-Status（GMII是基本寄存器）

# 剩余寄存器-厂商自定义
