# 第六章: RDMA CM 抓包分析

rdma_cm（RDMA Connection Manager）是一个标准化的连接管理库，提供类似 socket 的 API 来建立 RDMA 连接，屏蔽底层细节。

perftest 工具默认使用 tcp 进行连接管理，但是可以通过 `--rdma_cm` 参数切换到 CM 模式。

继续上一章的测试，实验环境依旧不变，将 `10.1.16.61` 作为服务端，将 `10.1.16.62` 作为客户端。

[pcap抓包文件](../pcap/rdma_cm.pcap)

## 6.1 带内 CM 和带外 tcp 建连的区别

| 对比项       | TCP 带外方式                                         | CM 带内方式                                        |
| ------------ | ---------------------------------------------------- | -------------------------------------------------- |
| 连接建立方式 | TCP 连接交换参数                                     | 标准 CM 协议（RoCEv2 为 CM over UDP，IB 为 IB CM） |
| 参数交换内容 | QPN、PSN、RKey、GID 等                               | 标准化 CM 消息，包含地址、服务信息等元数据         |
| QP 状态迁移  | 应用自己调用 `ibv_modify_qp()`                       | 由 `librdmacm` 自动管理                            |
| 状态机流程   | `RESET → INIT → RTR → RTS`                           | 对应用透明，库内部完成                             |
| 编程接口     | 完全手动实现，需自行编写握手代码                     | 类 Socket API，只需 `listen/connect/accept`        |
| 协议标准化   | 私有实现，不同应用之间格式可能不同                   | 标准协议，各 RDMA 应用可互通                       |
| 典型应用     | perftest、NCCL、部分 MPI 实现、很多 HPC/AI 框架      | NVMe-oF、SRP、iSER、部分 MPI 实现                  |
| 消息格式     | 通常是明文文本或自定义二进制格式                     | 结构化字段，包含 GUID、Service ID 等丰富元数据     |
| 连接生命周期 | TCP 连接通常在数据传输期间保持存在（也可以主动关闭） | 支持标准的连接管理和断连流程                       |
| 断开连接     | 应用自行决定如何退出和清理资源                       | 提供标准的 `DREQ/DREP` 断连握手                    |

> TCP 带外方式本质上是一套“自己造轮子”的连接管理方案，而 `rdma_cm` 则提供了标准化的连接建立、地址解析和断连机制，将应用从复杂的 QP 状态管理中解放出来。

---

## 6.2 测试环境准备

perftest 换成 `--rdma_cm` 参数，直接复用之前的测试框架：

```bash
# 服务端抓包（只抓 4791，不需要 18515）
sudo tcpdump -i ens160 -w /tmp/rdma_cm.pcap 'udp port 4791'

# 服务端
ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000 --rdma_cm

# 客户端
ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000 --rdma_cm 10.1.16.61
```

## 6.3 perftest CM 案例时序全貌

![rdma_cm](../assets/rdma_cm.png)

rdma_cm 在 RoCEv2 环境下走的是 **RDMA CM over UDP**，端口 **4791**，和 RDMA 数据包用同一个端口。

通过分析 wireshark，可以得到如下时序全貌：

```bash
包 01  62→61 [UD CM] ConnectRequest  Local QPN=0x3d    ← 建控制通道
包 02  62←61 [UD CM] ConnectReply    Local QPN=0x3b
包 03  62→61 [UD CM] ReadyToUse

包 04  62→61 [RC] RDMA Send × N                          ← 控制通道交换测试参数 (VAddr, RKey...)

包 24  62→61 [UD CM] ConnectRequest  Local QPN=0x3e    ← 建数据通道
包 25  62←61 [UD CM] ConnectReply    Local QPN=0x3c
包 30  62→61 [UD CM] ReadyToUse

包 43  62→61 [RC] RDMA Write × N                       ← 正式测试开始

包 91  62→61 [CM] DisconnectRequest  Remote QPN=0x3c   ← 测试结束，先拆数据通道
包 92  62←61 [CM] DisconnectRequest  Remote QPN=0x3e   ← 拆数据通道
包 95  62←61 [CM] DisconnectRequest  Remote QPN=0x3d   ← 再拆控制通道
```

---

从抓包中，我们可以看到两次 ConnectRequest，它们是两条**独立的连接请求**，建的是两个不同的 QP。

```bash
                   包 1 (ConnectRequest)     包 24 (ConnectRequest)
Local Comm ID         0x64c0c56d               0x65c0c56d       ← 不同，各自独立的会话ID
Local QPN             0x00003d                 0x00003e         ← 不同！两个不同的 QP
IP CM Source Port     0xccf7                   0xc6be           ← 不同，两个独立的 CM 端口
```

两个 QP 的信息统计如下：

```
                  k8s-62 (Client)        k8s-61 (Server)
第一条连接:         QP 0x00003d  ←────────→  QP 0x00003b   (Send/Recv 控制通道)
第二条连接:         QP 0x00003e  ←────────→  QP 0x00003c   (实际数据通道)
```

**第一条连接（QP 0x3d ↔ 0x3b）：参数协商通道**

在正式测试开始前，perftest 需要在两端之间交换测试参数，包括 MR 的 VAddr、RKey、消息大小、迭代次数等。这些元数据通过 Send 消息传递，走的就是第一条连接。

**第二条连接（QP 0x3e ↔ 0x3c）：测试数据通道**

参数协商完成后，perftest 再建第二条连接，用于跑实际的 RDMA Write 测试流量。

## 6.4 CM 协议解析

![Connect Request](../assets/rdma_cm_2.png)

从抓包中可以看到一个有趣的现象：CM 发起 ConnectRequest 的时候，用的是 UD（Unreliable Datagram），BTH 中的 Destination QPN 是 1，并且报文中携带了标准的 MAD（Management Datagram）。

这说明 CM 本身并不是应用层协议，而是 InfiniBand 管理平面的一部分。CM 消息属于一种特殊的 MAD，其 `Management Class` 固定为 `0x07`，表示 **Communication Management**。

从协议栈的角度来看，一条 CM 消息的封装关系如下：

```text
CM Message
    ↓
MAD (Management Class = 0x07)
    ↓
UD Transport
    ↓
QP1 (General Service Interface)
    ↓
InfiniBand / RoCEv2 Network
```

> QP1 是 InfiniBand 管理平面的公共服务通道（预留的、始终存在的），主要用于承载 SA、CM、PMA 等 Management Datagram（MAD）消息，例如路径查询、连接管理和性能管理等操作。普通应用程序的数据通信不应该使用 QP1。

CM 之所以使用 QP1 和 UD 传输，是因为此时真正的数据 QP 尚未建立。双方还不知道彼此的 QPN、PSN 等连接参数，自然也不可能通过 RC 或 UC 发送数据。因此，CM 必须先借助已经存在的管理通道（QP1）完成连接参数的协商，然后才能建立真正的数据通道。

从这个意义上说，CM 扮演的角色与 TCP 的三次握手非常类似：它负责建立一条 RDMA 连接，但并不关心连接建立之后应用究竟要传输什么数据。

我们可以把包 1 中 CM `ConnectRequest` 的重要信息做一个摘录：

```bash
# ConnectRequest（包1）
Local Communication ID: 0x64c0c56d
Local QPN:              0x00003d
Starting PSN:           0x7cc204
Primary Local GID:      10.1.16.62
Primary Remote GID:     10.1.16.61
Local CA GUID:          0x025056fffea77d03
Path MTU:               1024
```

> 把这个 ConnectRequest 翻译成通俗易懂的语言就是：“我的 QPN 是 0x3d，初始 PSN 是 0x7cc204...，希望和你建立一条 RC 连接。”

接着，再来看看服务端收到请求后，返回的 `ConnectReply`（包 2）：

```bash
# ConnectReply（包2）
Local Communication ID: 0x8988f456
Remote Communication ID: 0x64c0c56d
Local QPN:              0x00003b
Starting PSN:           0x727d6d
Local CA GUID:          0x025056fffea7129f
```

> 把这个 ConnectReply 翻译成通俗易懂的语言就是：“看到了你的连接请求，我的 QPN 是 0x3b，初始 PSN 是 0x727d6d...，希望和你建立一条 RC 连接。

最后，客户端返回 `ReadToUse`（包3）：

```bash
# ConnectReply（包2）
Local Communication ID: 0x64c0c56d
Remote Communication ID: 0x8988f456
```

至此，双方已经完成了：对端 QPN 的交换、初始 PSN 的交换，以及路径和能力参数的协商。

随后，`librdmacm` 会自动推进 QP 状态机（RESET → INIT → RTR(Ready To Receive) → RTS(Ready To Send)），当双方都进入 RTS 状态后，CM 的使命便宣告完成，后续的数据传输将完全由新建立的 RC QP 接管。

与带外 TCP 建连方式相比，带内 CM 双方交换的信息实际上几乎完全相同，区别仅在于：

- TCP 方式使用应用私有格式交换参数；
- CM 方式使用标准化的 MAD 消息和结构化字段。

这也是 NVMe-oF、iSER 等不同存储厂商实现能够互联互通的根本原因：它们遵循的是同一套标准化的 CM 协议，而不是各自定义一套私有的 TCP 握手格式。
