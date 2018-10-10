---
layout: post
title:  "《分布式系统》读书笔记 - 第三章 网络和网络互联"
date:   2018-10-4 03:00:00 +0800
categories: DistributedSystems
tags: DistributedSystems

---

## 第三章 网络和网络互联

主要为网路基础，OSI参考模型协议等。

- 应用层：应用要求，HTTP、FTP、SMTP、CORBA、IIOP
- 表示层：网络表示传输数据，TLS安全
- 会话层：故障检测、自动修复，SIP
- 传输层：处理消息的最低一层，TCP/UDP
- 网络层：在特定网络中的计算机之间传输数据包，IP、ATM虚电路
- 数据链路层：负责在有物理连接的结点间传送数据包，Ethernet MAC、PPP
- 物理层：模拟信号传输二进制数据序列，ISDN、Ethernet基带信号

但是其实在互联网中，没有清楚地区分应用层、表示层、会话层。

### 1.路由协议/路由算法

距离-向量算法

### 2.互联网协议

TCP/IP协议组

### 3.IP寻址

#### 3.1 IP结构
<br/>      
      
![](../../../images/article/ds3_1.png) 

#### 3.2 IP协议

IP协议将数据从一个主机传到另一个主机，如果需要的话还会经过中间的路由器。完整的IP数据包格式是相当复杂的。下图给出的了主要的部分。有一些头部域没有显示在图中，他们是用于传输和路由算法的。

![](../../../images/article/ds3_2.png) 

IP传输有**不可靠**和**尽力而为**的语义。因为没有传输上的保证，数据可能会丢失、重复、延迟、顺序错误。**IP中唯一的校验是头部的校验和，计算代价不高，还能保证检测到任何寻址和数据包管理数据中发生的错误**。但是他**没有对数据的校验和**，避免了经过路由器的开销。而我们一般会用更高层的协议（TCP/UDP）来提供他们自己的校验和。

IP层将IP数据报放入适合的底层网络（例如Ethernet）传输的网络数据包中。当IP数据报的长度大于底层网络的MTU时，就在发送端将IP数据报分割成多个小的数据包，然后在目的地重新组装。

#### 3.3 地址解析 （ARP）

地址解析模块负责将互联网地址转为特定底层网络所使用的网络地址（有时称为物理地址）。例如，如果底层网络是Ethernet，那么地址解析模块将把32比特的互联网地址转换成48比特的Ethernet地址。

每个主机上的ARP模块会维护一个缓存，保存以前的（IP地址，Ethernet地址）对。

如果需要的IP在缓存内，请求可以立即应答。如果没有，ARP模块会在本地的Ethernet上发出一个Ethernet广播数据包（ARP数据请求包），数据包中包括了所需的IP地址。

本地Ethernet的每个计算机都收到这个请求，并且用自己的IP地址和数据包中的IP进行匹配。

如果匹配，就给ARP请求的发出方发送一个ARP应答，应答中包括自己的Ethernet地址。  
如果不匹配，就忽略该数据包。

发出方的ARP模块在自己的本地（IP地址，Ethernet地址）缓存中加入新的IP地址到以太网地址的映射表，这样将来如果响应类似的ARP请求就不需要在广播了。一段时间之后，每个计算机的ARP缓存中都包含了所有计算机的（IP地址，以太网地址）对。这时只有在新计算机加入到本地以太网时才需要ARP广播。

这里的理解借用segmentfault的问答:  
[TCP/IP: 在广域网（外网）上传输数据时会用到ARP协议吗](https://segmentfault.com/q/1010000006194417)

- 问题1：那如果在外网上，两个公网IP（A和B）互发数据，这个时候ARP还需要吗？ARP做广播的话明显不可能，范围太广了，而且广播应该只是在局域网做的。

- 问题2：如果是一个A机器（外网ip）要给一个B机器（内网ip中的某一台机器）发送数据的话呢，需要ARP协议吗？换句话说，需要知道B的MAC地址吗？

- 问题3：一个路由管辖着一个网段（内网），对外的话这个路由其本身是有一个公网ip的，外网要向内网中的机器发数据，它只知道路由的公网ip，并不知道内网中电脑的ip，那如果上述两个问题要用到ARP协议的话，那么要得知一个内网ip的机器的MAC地址，这个时候是不是就要用到路由器的NAT地址转换功能了？

- 问题4：A向B发送以太网数据帧（ARP请求）的时候，以太网数据帧（含ip数据报）（ps：这两个以太网数据帧是不同的）是处于待发状态（还在A中），还是说已经到达B的网段的路由器了？B机器的ARP应答是发给A机器还是发给路由器？

先明确两个事情：

- MAC地址是在链路层上使用的地址，也就是以太网数据帧上

- IP地址是在网络层上使用的地址，也就是TCP/IP帧。TCP/IP帧包装在以太网上面，所以当TCP/IP帧发送出去的时候需要由链路层再去确认MAC地址。

一个TCP包需要发送出去需要把它包装在以太网数据帧上面，也就意味着需要指明MAC地址，那么在还不知道目标主机MAC的情况下源主机就需要发送ARP请求来确定目的主机的MAC地址，在这段时间内以太网数据帧是没法发出去的（数据帧上的MAC都没设置怎么发？），也即等待（回答你的问题4）.

外网不能直接和内网的IP通信，如果需要通信需要使用到NAT，NAT用于映射内外网的IP、端口关系，而IP和端口都是在网络层上的内容，只讨论这个的话暂时没有和MAC地址有直接的关系，需要将网络层上的数据通过以太网数据帧发送出去的时候才有MAC和ARP的事情。（问题2、3）

广域网上的主机并不是全部并联起来的，他们被无数的路由器和交换机划分成不同的子网，他们之间的数据（大部分，除非刚好就在同一个子网内）需根据路由策略经过不同的路由一级一级进行进行转发之后才能送到目标主机，而这一个个的子网你也可以简单的把它理解成局域网，ARP的确会在这个“局域网”内广播，但并不像你想象的整个网络上进行广播。

无论是公网还是局域网，链路层上的通信都需要MAC地址，也就意味着也可能需要ARP来确认MAC地址。只是对于主干网而言，为了提高安全性和效率，一般会做MAC和IP的静态绑定，减少ARP的次数。

网络的最精明之处就是路由，通过复用相同的模式将无数的小局域网变成大局域网，再将无数的大局域网联成庞大的广域网，而这些网络间的通信则使用路由策略来控制。

### 4.TCP和UDP

#### UDP

UDP基本上是IP在传输层的一个复制。UDP数据被封装在一个IP数据包中，它具有一个包含了**源端口号**和**目的端口号**的短的头部（相应的主机地址位于IP头部）、一个长度域和一个校验和。UDP不提供传输保证。

IP数据包可能由于拥塞和网络错误被丢弃。除了可选的校验和外，UDP未增加任何额外的可靠性机制。如果校验和区域非零，则接收主机根据数据包内容计算出一个校验值，与接收到的校验和相比，若两者不匹配则数据包被丢弃。

UDP提供了一种在IP上附加最小开销或传输延迟、在进程对（或IP组播、一个进程对多个进程）之间传送最长达64KB的消息的方法。它不需要创建任何开销以及管理用的确认消息，但它适合于不需要可靠传送单个或多个消息的服务和应用

#### TCP

TCP是面向连接的，双向的，可靠的

**可靠性：**

- 排序：

	TCP发送进程将流分割成数据片断序列，然后将之作为IP数据包传送。每个TCP片断均有
一个序号。它在该片断的第一个宇节给出流中的字节数。接收程序在将数据放入接收进程的输入流前，
使用序号对收到的片断排序。只有所有编号较小的片断都已收到并且放入流中后，编号大的片断才能
被放入流中，因此，未按顺序到达的片断必须保存在一个缓冲区中，直到它前面的片断到达为止。

- 流控制：

	发送方管理不能使接收方或者中间结点过载，这通过片断确认机制完成。当接收方成功
地接收了一个片断后，它会记录该片断的序号。接收方会不时地向发送方发送确认倍息，给出输入流
中片断的最大序号以及窗口大小。如果有反向的数据流，则确认信息被包含在正常的片断中，否则被
放在确认数据片中。确认片断中的窗口大小域指定了在下一个确认之前允许发送方传送的数据量。


- 重传:

	发送方记录它发送的片断的序号。当它接收到一个确认消息时，它知道片断被成功接收，
并将之从外发缓冲区中清除。如果在一个指定超时时间内，片断并没有得到确认，则发送方重发该
片断。

- 缓冲:

	接收方的接收缓冲区用于平衡发送方和接收方之间的流量。如果接收进程发出receive 操作
的速度比发送进程发出send操作的速度慢很多，那么缓冲区中的数据量就会增加。通常情况下，数据
在缓冲区满之前被取出，但最终缓冲区会溢出，此时到达的片断不被记录就直接被丢弃了。因此，接
收方不会给出相应的确认，而发送方将被迫重新发送片断。

- 校验和:

	每个片断包含-一个对头部和片断中数据的校验和，如果接收到的片断和校验和不匹配,
则片断被丢弃。

#### DNS

域名 -> IP

#### 防火墙

防火墙由一组进程实现，它作为通向私有内部网的网关。

- 服务控制
- 行为控制
- 用户控制
- IP数据包过滤
- TCP网关
- 应用层网关