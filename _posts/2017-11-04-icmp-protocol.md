---
title: ICMP协议
date: 2017-11-04 14:31:16
categories:
- technique
tags:
- TCP/IP
- Network
---

## 前言
> 前面学习`ARP`和`RARP`的知识点，接着学习`ICMP`协议知识点。

## ICMP

> `ICMP`称为`Internet控制报文协议`。其常被认为是`IP`层的组成部分，它传递差错报文以及其他需要注意的信息，`ICMP报文`通常被`IP`层或更高层协议(`TCP`、`UDP`)使用，`ICMP`报文把差错报文返回给用户进程，其是在`IP`数据报内部传输，如下图所示。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-and-ip.png)

`ICMP`报文的格式如下所示，所有报文的前`4`个字节均一样，剩下字段互不相同。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-protocol.png)

* 类型字段有**15**个不同值，用来描述特定类型的`ICMP报文`。
* 代码字段用来进一步描述不同的条件。
* 检验和字段覆盖整个`ICMP`报文，其是必须的。

### ICMP报文类型

> 各类型`ICMP`报文如下图所示。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-1.png)
![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-2.png)

不同报文类型由报文中的**类型字段**和**代码字段**共同决定，最后的**查询**和**差错**表示报文是`查询报文`或者`差错报文`，对`差错报文`需要做特殊处理(如在对`ICMP`差错报文进行响应时，永远不会生成另一份`ICMP`差错报文，否则可能会永远无休止地循环)。当发送一份`ICMP`报文时，报文始终包含`IP`的首部和产生`ICMP`差错报文的`IP`数据报的**前8个字节**。这样接收`ICMP`差错报文的模块就会把它与某个特定的协议(`IP`数据报首部的协议字段)和用户进程(`IP`数据报前8个字节中的`TCP`或`UDP`报文首部中的`TCP`或`UDP`端口号)联系起来。下面的情况都不会导致产生`ICMP`差错报文。

* `ICMP`差错报文(查询报文可能会产生差错报文)。
* 目的地址是广播地址或多播地址的`IP`数据报。
* 作为链路层广播的数据报。
* 不是`IP`分片的第一片。
* 源地址不是单个主机的数据报(源地址不能为零地址、环回地址、广播地址、多播地址)。

> 上述规则均是为了避免可能引起的广播风暴。

### ICMP地址掩码请求与应答

> `ICMP`地址掩码请求用于无盘系统在引导过程中获取自己的子网掩码。

系统广播自己的`ICMP`请求报文(与使用`RARP`获取`IP`地址类似)；还可使用`BOOTP`协议获取子网掩码。请求和应答报文格式如下所示。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-address-mask.png)

* 请求中的类型为`17`，应答中的类型为`18`。
* 代码字段为`0`。
* 检验和字段用作检验报文。
* **标识符字段**和**序列号字段**由发送端任意选择设定，值在应答中被返回，这样即可匹配请求与应答。

### ICMP时间戳请求与应答

> `ICMP`时间戳请求允许系统向另一个系统查询当前的时间。

该报文提供了毫秒级的分辨率，使用其他方法从其他主机获取的时间只能提供秒级分辨率，其请求和应答报文格式如下所示。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-request-and-response.png)

* 请求端发送请求时填写发起时间戳。
* 应答端接收请求时填写接收时间戳。
* 应答端发送应答时填写发送时间戳。

### ICMP端口不可达差错

> 差错报文，即端口不可达，是`ICMP`目的不可达报文中的一种。

若`UDP`接收一份`UDP`报文而目的端口与某个正在使用的进程不相符，那么`UDP`返回一个`ICMP`不可达报文，可使用`TFTP`强制生成一个端口不可达报文，如下图所示为`UDP`端口不可达的`ICMP`报文格式。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/udp-unreachable.png)

`ICMP`差错报文必须包括生成该差错报文的数据报`IP`首部(包含协议字段)以及跟在该`IP`首部后的前`8`个字节(包含源端口和目的端口)。`IP`首部中包含了协议字段，可使得`ICMP`可以知道如何解析后面的`8`个字节。`ICMP`不可达报文格式如下。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/icmp-unreachable.png)

`ICMP`不可达报文类型为`3`，总共有`16`种类型(0-15)。

### ICMP报文的BSD处理

在每个系统中，对于`ICMP`报文的处理各不相同，下图所示为`4.4BSD`对于`ICMP`报文的处理方法。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/tcpip/icmp/bsd-for-icmp.jpg)

可知对于`ICMP`报文而言，可被内核、用户进程、丢弃等处理方法。

## 总结

> 本篇讲解了不同`ICMP`报文类型，并着重讲解了地址掩码请求与应答、时间戳请求与应答以及端口不可达差错等报文，更偏向于理论知识，了解即可。



