# QUESTIONS ABOUT TCP/IP

> TCP, UDP区别

TCP（Transmission Control Protocol，传输控制协议）是面向连接的协议，也就是说，在收发数据前，必须和对方建立可靠的连接。 一个TCP连接必须要经过三次“对话”才能建立起来

- 面向连接的, 可靠的, 基于字节流的传输层通信协议
- 将应用层的数据流分割成报文段并发送给目标节点的TCP层
- 数据包都有序号, 对方收到则发送ACK确认, 未收到则重传

> TCP Flags (标志位)

- URG: 紧急指针标志
- ACK: 确认序号标志
- PSH: push标志
- RST: 重置连接标志
- SYN: 同步序号, 用于建立连接
- FIN: finish标志, 用于断开连接

> TCP三次握手过程  

- 第一次握手：建立连接时, 客户端发送SYN包(syn=j)到服务器, 并进入SYN_SEND状态, 等待服务器确认
- 第二次握手：服务器收到SYN包, 必须确认客户的SYN(ack=j+1), 同时自己也发送一个SYN(syn=k),即SYN+ACK包, 此时服务器进入SYN_RECV状态
- 第三次握手：客户端收到服务器的SYN+ACK包, 向服务器发送确认包ACK(ack=k+1), 此包发送完毕, 客户端和服务端进入ESTABLISHED状态, 完成三次握手

3次握手的特点: 没有应用层的数据, SYN这个标志位只有在TCP建立连接时才会被置1, 握手完成后SYN标志位被置0。

> TCP建立连接要进行3次握手，而断开连接要进行4次

1. 第一次： Client发送一个FIN, 用来关闭Client到Server的数据传送, Client进入FIN_WAIT_1状态
2. 第二次： Server收到FIN后, 发送一个ACK给Client, 确认序号为收到序号+1(与SYN相同, 一个FIN占用一个序号), Server进入CLOSE_WAIT状态
3. 第三次： Server发送一个FIN, 用来关闭Server到Client的数据传送, Server进入LAST_ACK状态
4. 第四次： Client收到FIN后, Client进入TIME_WAIT状态, 接着发送一个ACK给Server, 确认序号为收到序号+1, Server进入CLOSED状态, 完成四次挥手

> UDP（User Data Protocol，用户数据报协议）

1. UDP是一个非连接的协议，传输数据之前源端和终端不建立连接， 当它想传送时就简单地去抓取来自应用程序的数据，并尽可能快地把它扔到网络上。 在发送端，UDP传送数据的速度仅仅是受应用程序生成数据的速度、 计算机的能力和传输带宽的限制； 在接收端，UDP把每个消息段放在队列中，应用程序每次从队列中读一个消息段。
2. 由于传输数据不建立连接，因此也就不需要维护连接状态，包括收发状态等， 因此一台服务机可同时向多个客户机传输相同的消息。
3. UDP信息包的标题很短，只有8个字节，相对于TCP的20个字节信息包的额外开销很小。
4. 吞吐量不受拥挤控制算法的调节，只受应用软件生成数据的速率、传输带宽、 源端和终端主机性能的限制。
5. UDP使用尽最大努力交付，即不保证可靠交付， 因此主机不需要维持复杂的链接状态表（这里面有许多参数）。
6. UDP是面向报文的。发送方的UDP对应用程序交下来的报文， 在添加首部后就向下交付给IP层。既不拆分，也不合并，而是保留这些报文的边界， 因此，应用程序需要选择合适的报文大小。

> 三次握手, 四次挥手, 为什么要四次挥手

- 为了初始化Sequence Number的初始值
- 因为全双工, 发送方和接收方都需要FIN和ACK报文

> 长连接和短连接

TODO
> 连接池适合长连接还是短连接

 连接池适合长连接
> http无状态?

无状态是指协议对于事务处理没有记忆能力，服务器不知道客户端是什么状态。即我们给服务器发送 HTTP 请求之后，服务器根据请求，会给我们发送数据过来，但是，发送完，不会记录任何信息。

> 首次握手的隐患(SYN FLOOD)

首次握手之后, 客户端离线了, 服务端需要等待63秒, 重试连接五次才断开连接(可能会耗尽SYN队列)

SYN队列满后, 通过tcp_syncookies参数回发SYN Cookie, 若为正常连接则Client会回发SYN Cookie, 直接连接

> 建立连接后, Client出现故障?

heart beat

> 为什么会有TIME_WAIT状态

- 确保有足够的时间让对方收到ACK包
- 避免新旧连接混淆

> TCP的滑动窗口

RTT: 发送一个数据包收到对应的ACK所花费的时间  
RTO: 重传时间间隔

---
TCP使用滑动窗口做流量控制

> 输入 ping IP 后敲回车，发包前会发生什么？

ping目标ip时，先查路由表，确定出接口

- 如果落在直连接口子网内，此时若为以太网等 多路访问网络 则先查询arp缓存，命中则直接发出，否则在该接口上发arp询问目标ip的mac地址，取得后发出，若为ppp等 点对点网络 ，则直接可以发出；
- 如果查表落在缺省路由上，此时若为以太网等 多路访问网络 则先查询网关arp缓存，命中则直接发出，否则在该接口上发arp询问网关的mac地址，取得后发出，若为ppp等 点对点网络 ，则直接可以发出；
- 若查表未命中，则返回不可达。
