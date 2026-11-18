# socket连接流程

[TOC]

## socket流程

![socket连接流程](./img/socket连接流程.jpg)

## 关键接口

`socket`使用时有几个常用的接口

### socket函数

`socket()`函数的原型如下，这个函数建立一个协议族为`domain`、协议类型为`type`、协议编号为`protocol`的套接字文件描述符。如果函数调用成功，会返回一个标识这个套接字的文件描述符，失败的时候返回`-1`

**函数原型**

```c
int socket(int domain, int type, int protocol);
```

**domain参数**
函数`socket()`的参数`domain`用于设置网络通信的域，函数`socket()`根据这个参数选择通信协议的族。通信协议族在文件`sys/socket.h`中定义

**type参数**
函数`socket()`的参数`type`用于设置套接字通信的类型，主要有`SOCKET_STREAM`（流式套接字）、`SOCK——DGRAM`（数据包套接字）等

并不是所有的协议族都实现了这些协议类型，例如，`AF_INET`协议族就没有实现`SOCK_SEQPACKET`协议类型

**`protocol`参数**
函数`socket()`的第3个参数`protocol`用于制定某个协议的特定类型，即`type`类型中的某个类型。通常某协议中只有一种特定类型，这样`protocol`参数仅能设置为`0`；但是有些协议有多种特定的类型，就需要设置这个参数来选择特定的类型

**示例**

建立一个流式套接字：

```perl
int sock = socket(AF_INET, SOCK_STREAM, 0);
```

### bind函数

`bind`函数的作用是通过绑定一个其他`func`函数生成一个依赖于`func`的新的函数对象，复用`func`函数的实现，但是可以改变这个`func`的参数数量和顺序，可以绑定普通函数、全局函数，静态函数，成员函数，而且其参数可以支持占位符（std::placeholders::_1，std::placeholders::_2）来改变参数的顺序，并且可以设置`func`中默认的几个参数来减少输入参数的数量

**函数原型**

```c
int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
```

* `sockfd`：套接字描述符，是通过 `socket()` 创建的套接字。这个套接字将被绑定到服务端的地址和端口
    - 这里的 `addr` 是本地地址（本地IP和端口），将 `socket` 绑定到本地的某个 `IP` 地址和端口号
    - **用于服务器端，指定本地监听的地址和端口**
* `addr`：指向一个 `struct sockaddr` 结构体的指针，包含了套接字要绑定到的地址信息。对于 `IPv4`，通常是一个 `struct sockaddr_in` 结构体
* `addrlen`：地址结构的大小，通常使用 `sizeof(struct sockaddr_in)` 来表示

**返回值**

- **成功**：返回 `0`，表示绑定成功
- **失败**：返回 `-1`，并设置 `errno` 来指示错误原因

### listen函数

`listen`函数的作用是将一个套接字标记为监听状态，准备接受客户端的连接请求。它通常在创建并绑定一个服务器端套接字后使用，表示服务器愿意接受来自客户端的连接

**函数原型**

```c
int listen(int sockfd, int backlog);
```

* `sockfd`：这是一个套接字描述符，通常是通过 `socket()` 函数创建的

* `backlog`：这是一个整数，指定连接请求队列的最大长度。即，当客户端连接请求到来时，系统会将这些连接排队，直到 `accept()` 函数接受连接。如果队列已满，新的连接可能会被拒绝

**返回值**

- **成功**：返回 `0` 表示
- **失败**：返回 `-1` 表示出错，并且设置了 `errno` 以指示错误原因

### accept函数

`accept`函数是用于在套接字上接受传入连接的函数。它通常用于服务器端，以处理客户端的连接请求

**函数原型**

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- `sockfd`：服务器端的套接字描述符，通常是通过 `socket()` 函数创建并且通过 `bind()` 绑定后，再通过 `listen()` 设置为监听状态的套接字
- `addr`：指向一个 `struct sockaddr` 结构体的指针，用来存储客户端的地址信息（例如，IP 地址和端口号）。可以设置为 `NULL`，如果不关心客户端地址信息
- `addrlen`：指向一个变量，表示 `addr` 缓存区的长度，`accept()` 调用后，这个变量会被更新为客户端地址的实际长度

**返回值**

- **成功**：`accept()` 返回一个新的套接字描述符，表示与客户端的连接
- **失败**：返回 `-1`，并设置 `errno` 来指示错误原因

### connect函数

`connect`函数的功能可以用一句话来概括，就是完成面向连接的协议的连接过程，它是主要连接的。面向连接的协议，在建立连接的时候总会有一方先发送数据，那么谁调用了`connect`谁就是先发送数据的一方

**函数原型**

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

* `sockfd`：客户端的套接字描述符，通常是通过 `socket()` 函数创建的
    - 这里的 `addr` 是远程地址（目标服务器的`IP`和端口），将 `socket` 连接到远程主机的指定地址和端口
    - **用于客户端，指定要连接的服务器地址和端口**
* `addr`：指向 `struct sockaddr` 结构体的指针，包含了目标服务器的地址信息（`IP` 地址和端口）
* `addrlen`：地址结构的大小，通常可以使用 `sizeof(struct sockaddr_in)`

**返回值**

- **成功**：返回 `0` 表示连接成功
- **失败**：返回 `-1`，并设置 `errno` 以指示错误原因

### send函数

`send`函数通常用于 客户端 或 服务器 向对方发送消息或数据

**函数原型**

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

* `sockfd`：套接字描述符，是已建立连接的套接字（例如，通过 `socket()` 创建并通过 `connect()` 或 `accept()` 连接后的套接字）。它指定了要发送数据的目标套接字。
* `buf`：指向要发送的数据的缓冲区（内存地址）。这是一个指向 `char` 类型数据的指针，存储了需要发送的消息或数据
* `len`：要发送的数据的字节数，通常是缓冲区中数据的长度
* `flags`：设置发送行为的标志位。常用的标志包括：
    * `0`：表示正常发送
    * `MSG_CONFIRM`：请求确认消息发送
    * `MSG_DONTROUTE`：不使用路由表直接发送数据（绕过路由器）
    * `MSG_NOSIGNAL`：防止发送 `SIGPIPE` 信号

**返回值**

- **成功**：返回发送的字节数（即成功发送的数据量）。这个值可能小于 `len`，因为数据可能未完全发送，这时候需要再次调用 `send()` 继续发送剩余的数据
- **失败**：返回 `-1`，并设置 `errno` 来指示错误原因

### recv函数

`recv()` 函数用于从已经建立连接的套接字接收数据。它是 接收数据 的核心函数，通常在服务器端用于接收客户端发送的数据，或者在客户端接收从服务器传来的响应数据

**函数原型**

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

* `sockfd`：已连接的套接字描述符。这个套接字是通过 `socket()` 创建并通过 `connect()`（客户端）或 `accept()`（服务器端）建立连接的套接字。
* `buf`：指向接收数据缓冲区的指针，用于存储接收到的数据
* `len`：要接收的最大字节数。`recv()` 函数会将最多 `len` 字节的数据存入 `buf` 中
* `flags`：指定接收操作的行为标志。常用的标志有：
    * `0`：默认行为，正常接收
    * `MSG_PEEK`：查看缓冲区的数据而不将其从缓冲区中移除
    * `MSG_WAITALL`：要求 `recv()` 等到 `len` 字节的数据都被接收到（对于流套接字，可能需要多次调用才能完全接收 `len` 字节）

**返回值**

- **成功**：返回实际接收的字节数（可以小于请求的 `len`，特别是在流套接字（如 `TCP`）中）。如果返回 `0`，表示连接已关闭
- **失败**：返回 `-1`，并设置 `errno` 来指示错误原因

### close函数 & shutdown函数

#### close函数

`close` 一个套接字的默认行为是把套接字标记为已关闭，然后立即返回到调用进程

**函数原型**

```c
int close(int sockfd);     //返回成功为0，出错为-1.
```

#### shutdown函数

**函数原型**

```c
int shutdown(int sockfd,int howto);  //返回成功为0，出错为-1
```

该函数的行为依赖于`howto`的值

* `SHUT_RD`：值为`0`，关闭连接的读这一半

* `SHUT_WR`：值为`1`，关闭连接的写这一半

* `SHUT_RDWR`：值为`2`，连接的读和写都关闭

**两函数的区别**

1. `close`函数会关闭套接字`ID`，如果有其他的进程共享着这个套接字，那么它仍然是打开的，这个连接仍然可以用来读和写，并且有时候这是非常重要的 ，特别是对于多进程并发服务器来说
2. `shutdown`会切断进程共享的套接字的所有连接，不管这个套接字的引用计数是否为零，那些试图读得进程将会接收到`EOF`标识，那些试图写的进程将会检测到`SIGPIPE`信号，同时可利用`shutdown`的第二个参数选择断连的方式

## socket()函数参数详解

`socket()`函数的用户态`API`形态如下

```c
int socket(int family, int type, int protocol);
```

**参数含义**

| 参数       | 含义                                                         |
| :--------- | :----------------------------------------------------------- |
| `family`   | 地址族，如 `AF_INET`、`AF_INET6`、`AF_UNIX`、`AF_NETLINK`    |
| `type`     | 套接字类型，如 `SOCK_STREAM`、`SOCK_DGRAM`；可按位或 `SOCK_CLOEXEC` / `SOCK_NONBLOCK` |
| `protocol` | 协议号，通常填 `0` 表示默认                                  |

`family` 选地址族，`type` 选通信语义，`protocol` 在该地址族里再选具体协议。`protocol` 填 `0` 时，内核按 `family` + `type` 选默认协议

定义分别在 `include/linux/socket.h`（`AF_*`）、`include/linux/net.h`（`SOCK_*`）、`include/uapi/linux/in.h`（`IPPROTO_*`）。`PF_*` 与 `AF_*` 数值相同，是同一组常量的别名

### family

`family` 是地址族（`address family`），决定地址格式和协议栈。当前 `AF_MAX` 为 `46`，取值如下

| 值   | 宏                        | 含义                                                   |
| :--- | :------------------------ | :----------------------------------------------------- |
| `0`  | `AF_UNSPEC`               | 未指定                                                 |
| `1`  | `AF_UNIX` / `AF_LOCAL`    | 本机 `Unix` 域套接字                                   |
| `2`  | `AF_INET`                 | `IPv4`                                                 |
| `3`  | `AF_AX25`                 | 业余无线电 `AX.25`                                     |
| `4`  | `AF_IPX`                  | `Novell IPX`                                           |
| `5`  | `AF_APPLETALK`            | `AppleTalk`                                            |
| `6`  | `AF_NETROM`               | 业余无线电 `NET/ROM`                                   |
| `7`  | `AF_BRIDGE`               | 网桥                                                   |
| `8`  | `AF_ATMPVC`               | `ATM` 永久虚电路                                       |
| `9`  | `AF_X25`                  | `X.25`                                                 |
| `10` | `AF_INET6`                | `IPv6`                                                 |
| `11` | `AF_ROSE`                 | 业余无线电 `ROSE`                                      |
| `12` | `AF_DECnet`               | `DECnet`                                               |
| `13` | `AF_NETBEUI`              | `NetBEUI`（保留）                                      |
| `14` | `AF_SECURITY`             | 安全回调伪地址族                                       |
| `15` | `AF_KEY`                  | `PF_KEY` 密钥管理                                      |
| `16` | `AF_NETLINK` / `AF_ROUTE` | 内核与用户态的 `netlink`；`AF_ROUTE` 是 `BSD` 兼容别名 |
| `17` | `AF_PACKET`               | 链路层原始帧                                           |
| `18` | `AF_ASH`                  | `Ash`                                                  |
| `19` | `AF_ECONET`               | `Acorn Econet`                                         |
| `20` | `AF_ATMSVC`               | `ATM` 交换虚电路                                       |
| `21` | `AF_RDS`                  | `Reliable Datagram Sockets`                            |
| `22` | `AF_SNA`                  | `SNA`                                                  |
| `23` | `AF_IRDA`                 | 红外                                                   |
| `24` | `AF_PPPOX`                | `PPPoE / L2TP` 等                                      |
| `25` | `AF_WANPIPE`              | `Wanpipe`                                              |
| `26` | `AF_LLC`                  | `IEEE 802.2 LLC`                                       |
| `27` | `AF_IB`                   | `InfiniBand`                                           |
| `28` | `AF_MPLS`                 | `MPLS`                                                 |
| `29` | `AF_CAN`                  | `CAN` 总线                                             |
| `30` | `AF_TIPC`                 | 集群内通信                                             |
| `31` | `AF_BLUETOOTH`            | 蓝牙                                                   |
| `32` | `AF_IUCV`                 | `z/VM IUCV`                                            |
| `33` | `AF_RXRPC`                | `RxRPC`                                                |
| `34` | `AF_ISDN`                 | `mISDN`                                                |
| `35` | `AF_PHONET`               | `Phonet`                                               |
| `36` | `AF_IEEE802154`           | `IEEE 802.15.4`                                        |
| `37` | `AF_CAIF`                 | `CAIF`                                                 |
| `38` | `AF_ALG`                  | 内核加密算法接口                                       |
| `39` | `AF_NFC`                  | `NFC`                                                  |
| `40` | `AF_VSOCK`                | 虚拟机与宿主机通信                                     |
| `41` | `AF_KCM`                  | `Kernel Connection Multiplexor`                        |
| `42` | `AF_QIPCRTR`              | `Qualcomm IPC Router`                                  |
| `43` | `AF_SMC`                  | `Shared Memory Communications`，地址仍用 `AF_INET`     |
| `44` | `AF_XDP`                  | `AF_XDP`，用户态直接收发网卡帧                         |
| `45` | `AF_MCTP`                 | `Management Component Transport Protocol`              |

日常最常用的是 `AF_INET`、`AF_INET6`、`AF_UNIX`、`AF_NETLINK`、`AF_PACKET`。某个值能否用，还取决于内核是否编译了对应协议模块；未注册时 `socket()` 返回 `-EAFNOSUPPORT`

### type

`type` 的低 `4` 位是套接字类型，由 `SOCK_TYPE_MASK`（`0xf`）取出：

| 值   | 宏               | 含义                                      |
| :--- | :--------------- | :---------------------------------------- |
| `1`  | `SOCK_STREAM`    | 面向连接的字节流，如 `TCP`                |
| `2`  | `SOCK_DGRAM`     | 无连接数据报，如 `UDP`                    |
| `3`  | `SOCK_RAW`       | 原始报文，由调用者处理协议头              |
| `4`  | `SOCK_RDM`       | 可靠消息投递                              |
| `5`  | `SOCK_SEQPACKET` | 面向连接、保留消息边界                    |
| `6`  | `SOCK_DCCP`      | `Datagram Congestion Control Protocol`    |
| `10` | `SOCK_PACKET`    | 旧的设备层收包接口，已被 `AF_PACKET` 取代 |

低 `4` 位以外是创建标志，可与类型按位或：

| 宏              | 含义                                            |
| :-------------- | :---------------------------------------------- |
| `SOCK_CLOEXEC`  | 创建出的 `fd` 带 `O_CLOEXEC`，`exec` 时自动关闭 |
| `SOCK_NONBLOCK` | 创建出的 `fd` 为非阻塞，等同 `O_NONBLOCK`       |

例如 `SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK`

### protocol

`protocol` 的取值由 `family` 决定，没有一份全局枚举。`0` 表示该 `type` 的默认协议

`AF_INET` / `AF_INET6` 使用 `include/uapi/linux/in.h` 里的 `IP` 协议号：

| 值    | 宏                | 含义                               | 常见搭配                                          |
| :---- | :---------------- | :--------------------------------- | :------------------------------------------------ |
| `0`   | `IPPROTO_IP`      | 默认协议                           | `SOCK_STREAM` 得到 `TCP`，`SOCK_DGRAM` 得到 `UDP` |
| `1`   | `IPPROTO_ICMP`    | `ICMP`                             | `SOCK_RAW`                                        |
| `6`   | `IPPROTO_TCP`     | `TCP`                              | `SOCK_STREAM`                                     |
| `17`  | `IPPROTO_UDP`     | `UDP`                              | `SOCK_DGRAM`                                      |
| `33`  | `IPPROTO_DCCP`    | `DCCP`                             | `SOCK_DCCP`                                       |
| `41`  | `IPPROTO_IPV6`    | `IPv6` 封装                        | 隧道                                              |
| `47`  | `IPPROTO_GRE`     | `GRE`                              | `SOCK_RAW`                                        |
| `50`  | `IPPROTO_ESP`     | `IPsec ESP`                        | `SOCK_RAW`                                        |
| `51`  | `IPPROTO_AH`      | `IPsec AH`                         | `SOCK_RAW`                                        |
| `58`  | `IPPROTO_ICMPV6`  | `ICMPv6`                           | `AF_INET6` + `SOCK_RAW`                           |
| `132` | `IPPROTO_SCTP`    | `SCTP`                             | `SOCK_STREAM` 或 `SOCK_SEQPACKET`                 |
| `136` | `IPPROTO_UDPLITE` | `UDP-Lite`                         | `SOCK_DGRAM`                                      |
| `255` | `IPPROTO_RAW`     | 原始 `IP` 包，内核不自动加 `IP` 头 | `SOCK_RAW`                                        |
| `262` | `IPPROTO_MPTCP`   | `Multipath TCP`                    | `SOCK_STREAM`                                     |

同一文件里还有 `IPPROTO_IGMP`、`IPPROTO_PIM`、`IPPROTO_L2TP`、`IPPROTO_MPLS`、`IPPROTO_SMC` 等，数值就是 IP 头里的协议字段。

`AF_NETLINK` 的 `protocol` 是 `netlink` 协议号，定义在 `include/uapi/linux/netlink.h`，`type` 一般用 `SOCK_RAW`：

| 值   | 宏                       | 含义                            |
| :--- | :----------------------- | :------------------------------ |
| `0`  | `NETLINK_ROUTE`          | 路由、网卡、地址                |
| `2`  | `NETLINK_USERSOCK`       | 用户态之间通信                  |
| `4`  | `NETLINK_SOCK_DIAG`      | `socket` 诊断                   |
| `6`  | `NETLINK_XFRM`           | `IPsec` 策略                    |
| `9`  | `NETLINK_AUDIT`          | 审计                            |
| `11` | `NETLINK_CONNECTOR`      | 进程事件等                      |
| `12` | `NETLINK_NETFILTER`      | `netfilter`                     |
| `15` | `NETLINK_KOBJECT_UEVENT` | 设备热插拔事件                  |
| `16` | `NETLINK_GENERIC`        | `generic netlink`，如 `nl80211` |

`AF_PACKET` 的 `protocol` 是以太网类型（`EtherType`），并且要用网络字节序，例如 `htons(ETH_P_IP)`、`htons(ETH_P_ALL)`。

`AF_UNIX` 没有下层协议，`protocol` 填 `0`

`AF_BLUETOOTH`、`AF_CAN`、`AF_ALG`、`AF_VSOCK`、`AF_XDP` 各自有独立的协议常量，定义在对应的 `include/uapi/linux/*.h` 里

## 参考文档

> [Socket到底是什么？你想知道吗？-socket是什么意思 (51cto.com)](https://www.51cto.com/article/742745.html)
>
> [wiki百科socket]https://zh.wikipedia.org/wiki/%E7%B6%B2%E8%B7%AF%E6%8F%92%E5%BA%A7
>
> [Release v7.3 · torvalds/linux · GitHub](https://github.com/torvalds/linux/releases/tag/v7.3)

