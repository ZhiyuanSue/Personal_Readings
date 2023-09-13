Reconciliation子层将MII映射到PLS服务原语

# PLS服务原语对应的MII信号

## 1、PLS_DATA.request(OUTPUT_UNIT)

总共三个信号，1，0，以及DATA_COMPLETE，TXD的四位分别表示一个0或者1，从3-0，一个序列，当TX_EN置位的时候就传递这个信息，当TX_EN deassertion的时候传递DATA_COMPLETE消息。用TX_CLK传递同步消息。

### TX_ER（transmit coding error）

### TXD<3:0>（transmit data）

### TX_EN（transmit enable）

### TX_CLK（transmit clock）

这个是时钟信号，由PHY提供，因为TXD总共4位，所以，PHY提供的时钟信号应当是正常传输数据信号的1/4。比如100Mbps，就应当提供25MHz的一个信号。



这几个信号的组合有几种可能，然后会有不同的结果，具体见Table22-1（709页左右）

里面有个LPI的情况，感觉是低功耗状态（？）

## 2、PLS_SIGNAL.indication

### COL

包括signal_error和no_signal_error

col置位的时候表示error，反过来则no error

COL也不需要同步，0.8寄存器位置位时，行为未指定

## 3、PLS_DATA.indication(INPUT_UNIT)

这个只有0和1，由RXD四位表示，同样从3-0分别表示

### RXD<3:0>（receive data）

对RX_DV置位的每个RX_CLK指示的周期，将四位数据从PHY传输到reconciliation 子层。

### RX_ER

### RX_CLK（receive clock）

这个时钟信号同样由PHY提供，（因为接收数据的时钟并不完全对齐），所以PHY可以从接收的数据中回复RX_CLK基准，也可以从TX_CLK等时钟基准中获取RX_CLK时钟。

这里面没有了TX_EN，也就是说，传送这件事情需要有包结束信号，但是接收这件事情不需要。



同样的，和TX一样(在TX_EN不置位的时候)，RX_DV不置位的时候，用于指示LPI等等相关信号。

## 4、PLS_CARRIER.indication

### CRS

包括两个信号: carrier_on和carrier_off

如果接收和发送介质非空闲，CRS应当由PHY置位，如果都空闲则取消。

这个信号不需要与TX_CLK或者RX_CLK同步

如果控制寄存器的0.8也就是双工模式位置位，或者自协商选择了全双工模式，那么CRS的行为未指定。

## 5、PLS_DATA_VALUE.indication

### RX_DV（receive data valid）

在帧接收的时候，RX_DV和RX_ER如果都置位了，那么会指示这个帧有一个FrameCheckError。

包括两个值：DATA_VALUE和DATA_NOT_VALUE

这个我看了感觉还用于指示一些同步的问题。

# 其他

## MDC

MDC作为MDIO的时钟信号参考。

## MDIO

MDIO是PHY和STA之间的双向信号。



# MII数据流

数据包在MII上的传送应当具有以下的格式

< inter-frame >< preamble >< sfd >< data >< efd >

每个8位字节作为两个半字节进行发送和接收。

## Inter-frame

帧间周期不会发生任何数据活动，RX_DV信号取消和TX_EN信号取消置位表示不存在数据活动。

## Preamble

发送的情况，七个10101010

收包的情况，只能收到一个前导的10101011，这七个10101010没了。

## SFD

10101011




