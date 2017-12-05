---
title: Linux学习总结（九）——网络编程
date: 2016-12-18 11:52:36
categories: Linux学习记录
tags: [网络, TCP, UDP, 组播, 广播]
---

网络编程离不开的基础是网络体系结构，通常说的是 OSI协议参考模型，它是一个七层网络协议模型。而我们通常使用的 Internet 是基于 TCP/IP 协议的，TCP/IP 协议是 OSI 协议的一个4层简化模型，他们之间的对应关系如下图：
![网络协议模型](http://ogf054qp1.bkt.clouddn.com/OSI.png)

<!--more-->
自下而上，简单介绍下相应各层在整体架构中的作用:
 1. **网络接口层**：
    `Network Interface Layer` 是 TCP/IP 的最底层，负责将二进制转换为数据帧，并进行数据帧的发送和接收。
 2. **网络层**：
    `Internet Layer` 负责在主机之间的通信中选择数据包的传输路径。网络互连层定义了分组格式和协议，即 IP协议（Internet Protocol）。
 3. **传输层**：
    在 TCP/IP 模型中，传输层的功能是使源端主机和目标端主机上的对等实体可以进行会话。在传输层定义了两种服务质量不同的协议。
    即：传输控制协议 **TCP**（`transmission control protocol`）和用户数据报协议 **UDP**（`user datagram Protocol`）。
 4. **应用层**：
    TCP/IP 模型将 OSI 参考模型中的会话层和表示层的功能合并到应用层实现。应用层面向不同的网络应用引入了不同的应用层协议。其中，有基于 TCP 协议的，如文件传输协议（File Transfer Protocol，FTP）、虚拟终端协议（TELNET）、超文本链接协议（Hyper Text Transfer Protocol，HTTP），也有基于 UDP 协议的。

我们在这里主要讨论的是第三层传输层的具体内容，即 TCP 和 UDP。

## 一、网络编程基础知识
### 1. 套接字
套接字（Socket）是一种特殊的 I/O 接口，也是一种文件描述符。可以用于本地和网络通信，对于网络通信而言，每一个 Socket 都可以用网路地址结构（协议，本地地址，本地端口）来表示。Socket 通过一个特殊的函数创建，并返回一个整形的 Socket 描述符。
 > 套接字分为三种：
 * 流式套接字（SOCK_STREAM）：TCP 通信使用的就是流式套接字。
 * 数据包套接字（SOCK_DGRAM)：UDP 通信使用的就是数据包套接字。
 * 原始套接字（SOCKRAW）：原始套接字允许对底层协议进行直接的访问，主要用于协议的开发。

套接字有几个相关函数：

* 套接字的创建：
  ````C
  #include <sys/socket.h>
  int socket(int family, int type, int protocol);````
  功能：创建一个套接字。
  参数：`family` 指定相应的协议族，`type` 指定套接字类型，`protocol` 为0,(原始套接字除外)。
  返回值：成功返回非负的描述符，失败返回-1。

* 获取套接字选项：
  ````C
  #include <sys/types.h>
  #include <sys/socket.h>
  int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);````
  功能：获取套接字选项
  参数：`sockfd` 套接字描述符；`level` 选项所属协议层；`optname` 选项名称；`optval` 保存选项的缓存区；`optlen` 选项值长度。
  返回值：成功返回0，失败返回-1，并设置 errno。

* 设置套接字选项：
  ````C
  #include <sys/types.h>
  #include <sys/socket.h>
  int setsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);````
  功能：设置套接字选项
  参数：`sockfd` 套接字描述符；`level` 选项所属协议层；`optname` 选项名称；`optval` 保存选项的缓存区；`optlen` 选项值长度。
  返回值：成功返回0，失败返回-1，并设置 errno。

相应的套接字选项及说明如下：

|选项名称|说明|数据类型|
|---|-------|-------|
|**LEVEL**|**SOL_SOCKET**||
|SO_BROADCAST|允许发送广播数据报|int|
|SO_DEBUG|使能调试跟踪|int|
|SO_DONTROUTE|旁路路由表查询|int|
|SO_ERROR| 	获取待处理错误并消除	| 	int|
|SO_KEEPALIVE|周期性测试连接是否存活|int|
|SO_LINGER|若有数据待发送则延迟关闭	| 	linger{}|
|SO_OOBINLINE|让接收到的带外数据继续在线存放|int|
|SO_RCVBUF|接收缓冲区大小	 |	int|
|SO_SNDBUF|发送缓冲区大小	 |	int|
|SO_RCVTIMEO|接收超时	 |	timeval{}|
|SO_SNDTIMEO|发送超时	 |	timeval{}|
|SO_REUSEADDR|允许重用本地地址|int|
|SO_REUSEPORT|允许重用本地地址|int|
|SO_TYPE| 	取得套接口类型	 	|int|
|**LEVEL**|**IPPROTO_IP**||	
|IP_HDRINCL|IP头部包括数据|int|
|IP_OPTIONS|IP头部选项	| 	见后面说明|
|IP_TOS|服务类型和优先权|	 	int|
|IP_TTL|存活时间	| 	int|
|IP_ADD_MEMBERSHIP	 |加入多播组	 |	ip_mreq{}|
|IP_DROP_MEMBERSHIP	 |离开多播组	| 	ip_mreq{}|
|**LEVEL**|**IPPROTO_TCP**||	
|TCP_KEEPALIVE|控测对方是否存活前连接闲置秒数|	 	int|
|TCP_MAXRT|TCP最大重传时间	| 	int|
|TCP_MAXSEG|TCP最大分节大小| 	int|
|TCP_NODELAY|禁止Nagle算法|int|


### 2. IP/端口和网络字节序

IP 地址用来标识网络中的一台主机，端口用来标识主机内部的某个套接字。下面介绍几个相关的函数：

1. 地址格式转换函数：
   ````C
   #include <arpa/inet.h>
   int inet_addr(const char *strptr);                            
   int inet_pton(int family, const char *src, void *dst);
   char* inet_ntop(int family, void *src, char *dst, size_t len);````

   函数 `inet_addr()`/ `inet_pton` 用来要转换的字符串转换为 32 二进制IP地址（网络字节序）。函数 `inet_ntop` 是相应的反向操作。
   `family` 指的是地址族，用来区分 IPv4（AF_INET）和 IPv6（AF_INET6）。
   `inet_addr()` 成功返回相应的地址，失败返回-1；`inet_pton` 成功返回0，失败返回-1； `inet_ntop` 成功返回 dst，失败返回NULL。

2. 地址结构
   地址信息有两个相关的结构体：
   ````C
   struct sockaddr {
     unsigned short sa_family; //地址族
     char sa_data[14];         //14字节的协议地址
   }

   struct sockaddr_in {
     shor int sin_family;         //地址族
     unsigned short int sin_port; //端口号
     struct in_addr sin_addr;     //IP地址
     unsigned char sin_zero[8];   //填充0
   }````

   这两个数据类型大小相同，通常用 `sockaddr_in` 来保存某个网络地址，使用是强转成 `sockaddr`。`sa_family` 常见值有：
   * `AF_INET`  IPv4 协议
   * `AF_INET6` IPv6 协议
   * `AF_LOCAL` UNIX域协议
   * `AF_LINK` 链路地址协议
   * `AF_KEY` 密钥套接字

3. 网络字节序
   计算机的多字节整形存储方式有两种：大端（高位字节存储在地位地址）小端（高位字节存储在高位地址）。为了保证网络通信的一致性，数据以大端方式传输，所以需要相应的转换函数：
   ````C
   #include <netinet/in.h>
   uint16_t htons(uint16_t hostshort);
   uint32_t htonl(uint32_t hostlong);
   uint16_t ntohs(uint16_t netshort);
   uint32_t ntohl(uint32_t netlong);````


## 二、TCP 网络编程
TCP 向应用层提供可靠的面向链接的全双工数据流传输服务。它能提供高可靠性通信（数据无误，数据无丢失，数据无失序，数据无重复到达）。

### 1. 3次握手协议和两次挥手
TCP 的面向连接指的是：当计算机双方通信的时候必须先建立连接，然后进行数据通信，最后关闭连接。TCP 在建立连接时有三个步骤：
* **第一步** ： （客户端 -> 服务端）客户端向服务端发送一个包含 SYN 标志的 TCP 报文，并进入 SYN_SEND 状态，等待确认。
* **第二步** ： （服务端 -> 客户端）服务端在收到客户端的 SYN 报文之后，返回一个 SYN+ACK 的报文，表示服务端收到客户端的 SYN，服务端进入 SYN_RECV 状态。
* **第三步** ： （客户端 -> 服务端）客户端在收到服务端的 SYN+ACK 报文之后，向确认端发送确认 ACK 报文，客户端和服务端都进入 ESTABLISHED 状态。

在发送方发送一个数据包之后，会启动一个定时器，当数据包到达目的地之后，接收方会返回一个数据包，其中含有一个确认序号，如果发送方的定时器在确认信息到达之前超时，发送方会重发数据。

由于TCP连接是全双工的，因此每个方向都必须单独进行关闭。这个原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个 FIN只意味着这一方向上没有数据流动，一个TCP连接在收到一个FIN后仍能发送数据。首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。
* **第一步** 客户端A发送一个FIN，用来关闭客户A到服务器B的数据传送。 
* **第二步** 服务器B收到这个FIN，它发回一个ACK，确认序号为收到的序号加1。和SYN一样，一个FIN将占用一个序号。 
* **第三步** 服务器B关闭与客户端A的连接，发送一个FIN给客户端A。 
* **第四步** 客户端A发回ACK报文确认，并将确认序号设置为收到序号加1。  

### 2. TCP 的数据包头
![TCP 包头](http://ogf054qp1.bkt.clouddn.com/TCP%E5%8C%85%E5%A4%B4.png)
1. **源端口/目的端口**：16bit，标识本地和远端的端口号。
2. **顺序号**：32bit，标识发送的数据包的顺序。
3. **确认号**：32bit，希望收到的下一包数据的序列号。
4. **TCP头长**：4bit，表明 TCP 头中包含多少个32bit。
5. **6bit 未用**。
6. **URG**：表示紧急指针字段有效。
7. **ACK**：置位表示确认号字段有效；
8. **PSH**：表示当前报文需要请求推（push）操作；
9. **RST**：置位表示复位TCP连接；
10. **SYN**：用于建立TCP连接时同步序号；
11. **FIN**：用于释放TCP连接时标识发送方比特流结束。 
12. **窗口**：16位，表示源主机在请求接收端等待确认之前需要接收的字节数。它用于流量控制。
13. **校验位**：16位。用于检查TCP数据包头和数据的一致性。
14. **紧急指针**：16位。当URG码有效时只向紧急数据字节。
15. **可选项**：存在时表示TCP包头后还有另外的4字节数据。包括最大 TCP 载荷/窗口比例/选择重发数据包等选项。


### 3. 服务器端 TCP 网络编程：
TCP编程模型可分为服务器端和客户端两种，一个完整的服务器端 TCP 编程模型有6步：

1. 创建一个套接字 `socket()`
2. 连接到指定的地址和端口
    ````C
    #include <sys/socket.h>
    int bind(int sockfd, const struct sockaddr *myaddr, socket_t addrlen);````
    功能：将一个本地协议赋予一个套接字。
    参数：`sockfd` 套接字描述符，`myaddr` 指定的地址结构的指针，`addrlen` 该地址结构长度。
    返回值：成功返回0，失败返回-1；

3. 设置为监听模式
    ````C
    #include <sys/socket.h>
    int listen(int sockfd, int backlog);````
    功能：将一个未连接的套接字转换为一个被动套接字
    参数：`sockfd` 套接字描述符，`backlog` 相应的套接字排队的最大连接个数。
    返回值：成功返回0，失败返回-1；

4. 等待接收客户端的连接请求
    ````C
    #include <sys/socket.h>
    int accept(int sockfd, const struct sockaddr *cliaddr, socket_t addrlen);````
    功能：从已完成的链接队列队头返回下一个已完成连接，如果队列为空，进程进入睡眠（默认为阻塞模式）。
    参数：`sockfd` 套接字描述符，`cliaddr`/`addrlen` 用来存储对端的地址结构和长度
    返回值：成功返回非负的描述符，失败返回-1；

5. 发送和接收数据：`send()`/`recv()`
    ````C
    #include <sys/socket.h>
    int recvform(int sockfd, void *buff, size_t nbytes, int flags);
    int sendto(int sockfd, void *buff, size_t nbytes, int flags);````
    功能：类似标准的 `read()` 和 `write()` 函数。
    参数：前三个函数同 `read()`/`write()`， `flags` 一般是0。
    返回值：成功返回读或写的字节数，失败返回-1。

6. 关闭套接字 `close()`

### 4. 客户端的 TCP 编程模型
1. 创建一个套接字：`socket()`
2. 建立连接
    ````C
    #include <sys/socket.h>
    int connect(int sockfd, const struct sockaddr *servaddr, socket_t addrlen);````
    功能：建立与 TCP 服务器的连接
    参数：`sockfd` 套接字描述符，`servaddr`/`addrlen` 用来存储服务器端的地址结构和长度
    返回值：成功返回0，失败返回-1；
5. 发送和接收数据:`send()`/`recv()`
6. 关闭套接字 `close()`
  
## 三、UDP编程模型

UDP 即用户数据报协议，是一种面向无连接的不可靠的传输协议，具有消耗资源少，处理速度快的特点。
UDP 的数据包头比较简单，包含源和目的的地址（各16bit）8位空位，8位协议位，16位 UDP 长度。
UDP 网络编程和 TCP 网络编程的区别在于：UDP 是不可靠的数据报协议，客户端不与服务器建立连接，直接使用 `sendto()` 函数向服务器发送数据，而服务器同样不用接收连接，使用 `recvform` 返回相应的数据。
### 1. UDP 网络编程模型
* UDP 服务器端编程模型
  1. 创建一个套接字：同 TCP。
  2. 连接到指定的地址和端口 `bind()`
  5. 发送和接收数据 用 `sendto()`/`recvform()`
      ````C
     #include <sys/socket.h>
     int recvform(int sockfd, void *buff, size_t nbytes, int flags,
                  struct sockaddr *from, socklen_t *addrlen);
     int sendto(int sockfd, void *buff, size_t nbytes, int flags,
                  struct sockaddr *to, socklen_t *addrlen);````
     功能：类似标准的 `read()` 和 `write()` 函数。
     参数：前三个函数同 `read()`/`write()`， `flags` 一般是0， `from`/`to` 分别是对端的地址结构指针， `addrlen` 是对端的地址结构长度。
     返回值：成功返回读或写的字节数，失败返回-1。

  6. 关闭套接字 `close()`

* UDP 客户端编程模型
  1. 创建一个套接字：同 TCP。
  5. 发送和接收数据 用 `sendto()`/`recvform()`
  6. 关闭套接字 `close()`

### 2. 组播与广播编程
之前我们处理的都是单播程序：一个进程就与另一个进程通信。TCP 只支持单播寻址，而 UDP 还支持其他的寻址类型。

   * 单播（unicast）：向标识的单独接口递送数据，TCP 仅支持此种
   * 任播（anycast）：向标识的一组接口中的一个递送数据
   * 组播（multicast）：向标识的一组中的所有接口递送数据
   * 广播（broadcast）：向全体递送数据

广播和组播都需要使用UDP，都不能使用TCP。IPv4地址可以使用｛子网id，主机id｝来表示，-1表示所有位都为1的字段，广播可以分为几种： 

  * 子网定向广播地址，｛子网id，-1｝，指定子网上所有接口的广播地址
    > 192.168.1.0/24 该子网上的广播地址192.168.1.255 

  * 受限广播地址｛-1，-1｝即 `255.255.255.255`。

1. 广播的流程
  发送广播的流程是：
    * 创建 UDP 套接字
    * 指定目标地址和端口
    * 设置套接字选项允许发送广播包
      ````C
      setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on));````
    * 发送广播包
  
  接收广播包的流程是：
    * 创建 UDP 套接字
    * 绑定目标地址和端口
    * 接收广播包

2. 广播的缺点
  * 当使用单播时：
  发送端UDP 套接字承载了目的 IP，比如 192.168.32.3，和目的端口 比如 7433，在传输层，对他冠以一个 UDP 首部，在 IPv4 中标识为 17， 其次在网络层，标识为 IPv4, 确定了外出接口，接口层的以太网接口将相应的目的IP映射成 相应的以太网地址，并标识相应的帧类型为 IPv4（0x0800）。
  该数据在接口层传输，相应的主机接口首先判断该帧的以太网地址是否和本机一致：不一致忽略，一致读取整个帧，单播对非目的主机不造成任何额外开销。目的主机接口读取整个帧后，比较相应的帧类型 0x0800，传递给相应网络层协议 IPv4，网络层比较相应的 IP 是否和本机一致，一致后接收，随后查看相应的协议字段 17，数据被传送给传输层 UDP，UDP 查看相应的目的端口，把数据置于相应的套接字接收队列。

  * 当使用广播时：
  发送端UDP 套接字承载了目的 IP，比如 192.168.32.255，和目的端口 比如 7433，在传输层，对他冠以一个 UDP 首部，在 IPv4 中标识为 17， 其次在网络层，标识为 IPv4, 确定了外出接口，接口层的以太网接口将相应的目的IP映射成所在以太网的子网的以太网地址，此时这个以太网地址为全1的地址，并标识相应的帧类型为 IPv4（0x0800）。
  该数据在接口层传输，该子网内所有的主机接口都会判断该帧的以太网地址是和本机一致，读取整个帧，单播对非目的主机不造成任何额外开销。目的主机接口读取整个帧后，比较相应的帧类型 0x0800，传递给相应网络层协议 IPv4，网络层比较相应的 IP 是否和本机一致，一致后接收，随后查看相应的协议字段 17，数据被传送给传输层 UDP，UDP 查看相应的目的端口，如果有相应端口的应用进程，执行相应的接收程序，如果没有相应的进程，则 UDP 丢弃当前数据包。

  所以广播存在的问题是：子网上所有使用 IP 协议的主机都必须沿相应的协议传输到 UDP 层判断自己是否参加了相应的广播。所有不使用 IP 协议的主机也都必须在接口层接收所用的数据，并在网络层读取判断。

2. 组播地址
  IPv4 地址可以分为五类：
    * A 类：最高位0，主机号占 24 位，地址范围为：1.0.0.1 到 126.255.255.254
    * B 类：最高两位10，主机号占 16 位，地址范围为：128.0.0.1 到 191.255.255.254
    * C 类：最高三位110，主机号占 8 位，地址范围为：192.0.1.1 到 223.255.255.254
    * D 类：最高四位1110，地址范围为：224.0.0.1 到 239.255.255.254
    * E 类：保留
    其中，D类地址为组播地址，每一个组播地址代表一个多播组。

3. 组播的流程
  发送组播的流程是：
    * 创建 UDP 套接字
    * 指定目标地址和端口
    * 发送组播包

  接收组播的流程是：
    * 创建 UDP 套接字
    * 加入多播组
      ````C
      setsockopt(sockfd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));````
    * 绑定地址和端口
    * 发送组播包

## 四、本地套接字
套接字的引用本来就只支持本地通信，目前很多前台后台进程依旧使用 UNIX 域套接字进行通信，本地套接字的特点是使用简单，效率高。本地套接字也分为流式套接字和用户数据报两种类型。具体的编程方法和相应 TCP / UDP 套接字基本一致，区别仅是使用的协议和地址不同，这里就不再详细描述。

