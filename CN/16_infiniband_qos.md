# 第十六章：Infiniband Fabric 的 QoS

## 16.1 IB 里的 QoS

与其花很多篇幅介绍 QoS 定义，不如直接说说 QoS 要解决的问题：**当不同的流量同时争抢同一条链路时，谁先走、谁让路、谁能保证拿到多少带宽？**

QoS 问题在 AI 集群里变得非常具体，一条上行链路上可能同时跑着：

- 控制面的管理流量（例如：SM 的扫描）：量很小，但绝不能被饿死；
- 存储 I/O：对延迟敏感；
- 训练的集合通信（AllReduce / All-to-All）：带宽巨大，不折不扣的"大象流（elephant flow）"。

如果不加区分地让它们挤在一起，那条 AllReduce 的洪流会瞬间把链路缓冲塞满，管理流量和存储 I/O 只能干等。这种场景，任何一个做过以太网 QoS 的工程师都熟悉，**我们需要给流量分级，再按级别调度。**

所以这一章的目的，是帮助大家看清三件事：

- IB 如何解决队列与调度的问题？
- IB 如何让 QoS 与它那套信用流控机制配合在一起？
- IB 如何看待 QoS 标记的信任问题？

把这些想明白了，回过头再看以太网那套 DCB，也许会让大家有所启发：很多在以太网上靠一堆参数硬拼出来的能力，在 IB 里其实是被一体化设计好的。

---

## 16.2 两个核心概念：SL 与 VL

IB 的 QoS 建立在两个概念的分工之上。

### SL：贴在包上的"等级标签"

**SL（Service Level）是一个 4 bit 的字段，位于链路层头 LRH 里，由发送端设置，端到端保持不变。** 它能表达 16 个等级（SL0–SL15）。

SL 本身不是任何具体的缓冲区或队列，它只是一个抽象的等级标识，相当于以太网里的 802.1p PCP 或 DSCP。发送端说"我这股流量是 SL5"，这个标签就一路跟着包走，谁也不改它。

那么发送端怎么知道某股流量该用哪个 SL？

答案在控制平面：**应用通过 SA（Subnet Administration）查询 PathRecord 时，SA 会根据 QoS 策略为这条路径返回一个 SL。** 也就是说，"哪种流量用哪个 SL"是由 SM 的 QoS 策略统一规定、再通过 PathRecord 下发给端侧的，而不是应用自己拍脑袋定的。

### VL：真正的"物理队列"

**VL（Virtual Lane）才是真正的物理资源：每条 VL 在链路上拥有自己独立的一套缓冲区和独立的流控信用（credit）。** 多条 VL 复用同一条物理链路，但彼此的缓冲互相独立。这意味着一条 VL 被塞满，不会阻塞另一条 VL 的流量（避免队头阻塞，HoL blocking）。

> IB 的 credit 机制与 SAN 网络的 Buffer-to-Buffer Credit 思路相近，但粒度更细：SAN 的 BB credit 以链路为单位进行流控，整条端口共用一套信用池；而 IB 的 credit 则细化到每条 VL，每条虚拟通道独立持有自己的信用配额。

IB 规定：

- **VL0 ~ VL14**：最多 15 条数据 VL，供普通流量使用；
- **VL15**：保留给管理流量（SMP，子网管理包）专用，且**不受流控约束**。管理流量必须在任何拥塞下都能挤过去，所以它走一条永远不会被 credit 卡住的特殊通道。

不是所有设备都支持满 15 条数据 VL。一个端口支持几条，由 PortInfo 里的 **VLCap** 字段声明，常见取值是一个枚举：

| VLCap | Supported VL |
| ----- | ------------ |
| 1     | Only VL0     |
| 2     | VL0–VL1      |
| 3     | VL0–VL3      |
| 4     | VL0–VL7      |
| 5     | VL0–VL14     |

> 如果设备只支持 VL0 一条数据通道（VLCap=1），那么所有 SL 都只能映射到 VL0，QoS 的一切分级、调度都将退化成"只有一条队列"，形同虚设。
>
> 还要提前埋一个伏笔：VLCap 只是**硬件能力上限**。当前真正启用了几条 VL 是另一个字段 `OperVLs`，由 SM 按 `max_op_vls` 等参数决定，两者可以不相等。

### 为什么要拆成两个概念？

初看会觉得多此一举：为什么不让发送端直接指定 VL，非要绕一道 SL？

原因是 **SL 端到端不变，VL 却允许逐跳改变。** 一个包从源到目的可能跨越多条链路，每条链路支持的 VL 数量、用途规划可能都不一样。如果把"等级"和"具体走哪条物理通道"绑死，跨设备协调就会变成噩梦。

IB 的做法是解耦：**SL 是"我是什么等级"这个不变的意图，VL 是"在这一跳的这条链路上，我具体占用哪个缓冲"这个可变的实现。** 中间靠一张映射表把二者在每一跳挂接起来，这就是接下来要介绍的 SL2VL。

---

## 16.3 SL 到 VL 的映射（SL2VL）

既然 SL 在包里端到端不变，而 VL 要逐跳决定，那么每台设备在转发时就必须回答一个问题：**这个包标着 SL=X，我该把它放进出口链路的哪条 VL？**

回答这个问题的，就是 **SL2VL 映射表（SL to VL Mapping Table）**。

它的逻辑结构很简单，一张 16 项的查找表，把每个 SL 映射到一个 VL：

```
SL0  → VLa
SL1  → VLb
...
SL15 → VLx
```

有一个细节值得强调：**SL2VL 表是按"入端口 + 出端口"组织的**，即 `[IN][OUT]SL2VL` 各自是一张 16 项的 SL→VL 表。

为什么要细分到端口对？因为同一台交换机的不同链路可能支持不同数量的 VL，从某个口进、某个口出的流量，其 SL→VL 的合理映射可能并不相同。

终端 HCA 则相对简单，它只往外发包，不在端口之间转发，所以只需要一条映射即可。从显示上看，其 IN 和 OUT 端口恒定为 0，即 `[0][0]SL2VL` 。

于是，**一个包的完整旅程**是这样的：

1. 发送端标记 SL 发出；
2. 每经过一台交换机，交换机根据 DLID 查 LFT，确定出端口；
3. 交换机根据包里的 SL，查 `[IN][OUT]SL2VL`，决定这个包在下一条链路上进哪条 VL （SL 始终不变，VL 可能一跳一变）
4. 不断重复步骤 2 和 3，直到终端。

```bash
#           ┌──────── Sw1 ─────────┐
# port:     1      2       3       4
#           │      │       │       │
#          HcaA   HcaB    HcaC    HcaD
# LID:      2      5       8       9
```

举一个实际的例子，如上图所示，假设 HcaA(LID 2) 发给 HcaC(LID 8)，标记了 SL=10。这个包到达 SW1 时，交换机做两步独立的查表：

- 决定从哪个口出去，DLID=8：查 LFT──► 出端口 = 3
- 决定进哪条 VL，[IN=1][OUT=3]SL=10：查 SL2VL──► 某条 VL

### 用 VL 避免死锁

SL2VL 和 VL 还有一个超出"QoS"范畴的重要用途。

```bash
#        SwA ──────── SwB
#         │            │
#         │            │
#        SwD ──────── SwC
```

还记得第十二章介绍的那个 **credit loop（缓冲信用的循环等待）** 吗？我们当时是靠 updn 路由"禁止下行转上行"来打破环的。其实还有另一条解法：**在会形成环的那个转弯处，强制把包切换到另一条 VL 上去。** 因为不同 VL 的缓冲互相独立，循环依赖一旦被分散到不同的 VL，就再也合不拢——环被打破了。

OpenSM 里专为环面（torus）拓扑设计的 `torus-2QoS` 路由引擎，正是用"每个 QoS 等级配两条 VL"的办法来同时实现**无死锁**和**QoS 分级**。

所以，**VL 在 IB 里是一个身兼两职的资源：既服务于 QoS 分级，又服务于死锁规避。** 这一点和纯粹做 QoS 的以太网队列很不一样。

> 备注：现实中 IB 集群的主流拓扑是 fat-tree（各种 spine-leaf 变体），torus 属于少数派。因此 torus-2QoS 是一个非常小众的算法，这里就不展开了。

---

## 16.4 调度：VL 仲裁（VLArb）

SL2VL 解决了"包进哪条 VL"，但还有最后一个问题：**当同一个出端口上，好几条 VL 里都积压着包、都等着发出去时，端口先服务哪条 VL？**

这就是一个调度问题，在 IB 中被称为： **VL 仲裁（VL Arbitration）**，由 **VL 仲裁表（VLArbitration Table）** 控制。它同时提供了“严格优先级”和“加权公平”两种语义。

> 熟悉以太网的朋友们应该不陌生，差不多类似：优先级队列（PQ）和 基于类的加权公平队列（CBWFQ）。

仲裁表分成两张子表：

- **高优先级表（High Priority Table）**
- **低优先级表（Low Priority Table）**

每张子表都是一串 `(VL, 权重)` 的条目。权重以 **64 字节为单位**，表示该 VL 在一轮调度里可以连续发送多少个单位的数据，这本质上是一个**加权轮询（Weighted Round Robin）**。

两张表之间的关系是**严格优先级**：端口总是优先服务高优先级表里的 VL；只有高优先级表"额度用尽"后，才轮到低优先级表。而这个"额度",由一个关键字段限定：

- **VLHighLimit（高优先级上限）**：规定在被迫让位给低优先级之前，高优先级流量最多能连续发送多少。

**`VLHighLimit` 的存在是为了防止高优先级流量把低优先级彻底饿死**，再高的优先级，也不能无限霸占链路。`VLHighLimit` 可以通过 `smpquery PortInfo <port_ID>` 查询。

---

## 16.5 完整的工作机制

和以太网不一样，IB 的绝大部分配置都是由 OpenSM 下发的，QoS 也不例外，所有的 QoS 配置都由 SM 在子网初始化时算好、通过 SMP 下发到每个端口。工作机制大致如下：

```
QoS 策略（管理员定义）
      │  SM 计算
      ▼
每个端口的 SL2VL 表 + VLArb 表（SMP 下发）
      │
      ▼ 进每个包的转发：
  发送端按 PathRecord 给的 SL 打标
      → 交换机查 SL2VL，决定进哪条 VL
      → 出端口按 VLArb 表，决定 VL 间谁先发
```

---

## 16.6 实验测试

本章继续沿用上一那张单交换机拓扑：

```bash
#         ┌──────── Sw1 ─────────┐
#         │      │       │       │
#       HcaA   HcaB    HcaC    HcaD

Switch  8 "Sw1"
[1]     "HcaA"[1]
[2]     "HcaB"[1]
[3]     "HcaC"[1]
[4]     "HcaD"[1]

Hca     2 "HcaA"
[1]     "Sw1"[1]

Hca     2 "HcaB"
[1]     "Sw1"[2]

Hca     2 "HcaC"
[1]     "Sw1"[3]

Hca     2 "HcaD"
[1]     "Sw1"[4]
```

### 16.6.1 默认配置解读

分别在两个 terminal 中启动 ibsim 和 openSM：

```bash
# terminal 1：
rlwrap ibsim -s ./ib4.net

# terminal 2：
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local'
```

常用的查询 QoS 的相关命令如下：

```bash
# 查询端口信息
smpquery PortInfo (PI) <addr> [<portnum>]

# 查询SL2VL表
smpquery SL2VLTable (SL2VL) <addr> [<portnum>]

# 查询VL仲裁表
smpquery VLArbitration (VLArb) <addr> [<portnum>]
```

打开第三个 terminal，查看默认的 QoS 配置：

```bash
# 设置环境变量
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so


# 查询 HcaA port 1 信息（LID=2）
expert@net21:~$ smpquery pi 2 1
ibwarn: [18181] sim_connect: attached as client 1 at node "Sw1"
Lid:.............................2
SMLid:...........................1
...
VLCap:...........................VL0-7                  # 硬件能开 8 条数据 VL（VL0–VL7）
VLHighLimit:.....................0                      # 高优先级额度上限 = 0，意味着：当前所有流量都走低优先级仲裁
VLArbHighCap:....................8                      # 高优先级VL仲裁表最多能放 8 个 (VL,权重) 条目
VLArbLowCap:.....................8                      # 低优先级VL仲裁表最多能放 8 个 (VL,权重) 条目
MtuCap:..........................2048                   # 端口支持的最大 MTU（影响每条 VL 的缓冲尺寸，间接关系到 QoS）
VLStallCount:....................7                      # 连续因为队头超时被丢掉多少个包之后，这条 VL 被判定为"卡死（stalled）
HoqLife:.........................0                      # Head of Queue Lifetime，一个包在某条 VL 的队头最多能等多久
OperVLs:.........................VL0-1                  # 当前启用了 2 条 VL，只点亮了 VL0、VL1
...



# 查询 Sw1 port 1 信息（LID=1）
expert@net21:~$ smpquery pi 1 1
ibwarn: [18724] sim_connect: attached as client 1 at node "Sw1"
# Port info: Lid 1 port 1
Mkey:............................<not displayed>
Lid:.............................0
SMLid:...........................0
...
VLCap:...........................VL0-7
InitType:........................0x00
VLHighLimit:.....................0
VLArbHighCap:....................8
VLArbLowCap:.....................8
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................16
OperVLs:.........................VL0-1
...



# 查询 HcaA 的 SL2VL 表（Hca 的 IN 和 OUT 端口恒定为 0，即 `[0][0]SL2VL`）
expert@net21:~$ smpquery sl2vl 2
ibwarn: [18208] sim_connect: attached as client 1 at node "Sw1"
# SL2VL table: Lid 2
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
# 解释：因为 OperVLs=2，所以 SL 只能落到这两条上，于是 SL2VL 被算成 SL mod 2 = 0,1,0,1...



# 查询 Sw1 的 SL2VL 表（交换机默认 OUT 端口为 0，即SMA Port）
expert@net21:~$ smpquery sl2vl 1
ibwarn: [18210] sim_connect: attached as client 1 at node "Sw1"
# SL2VL table: Lid 1
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  1, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  2, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  3, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  4, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  5, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  6, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  7, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
ports: in  8, out  0: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14| 7|
# 解释：交换机管理流量（SMP，子网管理包）走的是 VL15，所以这张表实际只是个“默认的摆设”，实际不产生作用。



# 查询 Sw1 的 SL2VL 表（指定 OUT 端口为 1）
expert@net21:~$ smpquery sl2vl 1 1
ibwarn: [18521] sim_connect: attached as client 1 at node "Sw1"
# SL2VL table: Lid 1
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  1, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  2, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  3, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  4, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  5, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  6, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  7, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
ports: in  8, out  1: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
# 解释：因为 OperVLs=2，所以 SL 只能落到这两条上，于是 SL2VL 被算成 SL mod 2 = 0,1,0,1...



# 查询 HcaA port 1 的仲裁表
expert@net21:~$ smpquery vlarb 2 1
ibwarn: [18738] sim_connect: attached as client 1 at node "Sw1"
# VLArbitration tables: Lid 2 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x0 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# 解读：
# 以低优先级仲裁表为例：
# ┌──────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
# │ Col  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ VL   │  0  │  1  │  0  │  1  │  0  │  1  │  0  │  1  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ WEI  │  0  │  4  │  4  │  4  │  4  │  4  │  4  │  4  │
# └──────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
# OperVLs=2，只有两条 VL 可用，就拿它俩轮流填满 8 个槽
# VL0 累计权重 = 0+4+4+4 = 12，VL1 = 4+4+4+4 = 16。也就是 VL0 一轮发 12×64 字节、VL1 发 16×64 字节，大致 3:4，基本均衡、VL1 略多。
# 这里的权重以 64 字节为计量粒度；
# 所以，低优先级表 = VL0 和 VL1 大致平起平坐地加权轮询
#
# 以高优先级仲裁表为例：
# ┌──────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
# │ Col  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ VL   │  0  │  1  │  0  │  1  │  0  │  1  │  0  │  1  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ WEI  │  4  │  0  │  0  │  0  │  0  │  0  │  0  │  0  │
# └──────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
# 只有第一槽 (VL0, 4) 有权重，其余全是 0，看起来是"把 VL0 设成了高优先级"。
# 由于 VLHighLimit 这个"额度"目前设置为 0，即意味着高优先级表一个字节都发不了，整张高优先级表是失效的。




# 查询 Sw1 port 1 的仲裁表
expert@net21:~$ smpquery vlarb 1 1
ibwarn: [18740] sim_connect: attached as client 1 at node "Sw1"
# VLArbitration tables: Lid 1 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x0 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# 解读同上，不再赘述
```

### 16.6.2 OpenSM 配置选项

从本章开始，OpenSM 的配置选项越来越多了，不太方便继续采用类似前面几章的简单的命令行参数形式了。我们换一种生产环境的配置方法：将 OpenSM 的配置写入 `opensm.conf` 文件中，然后在启动 OpenSM 的时候加载。

如果希望了解 `opensm.conf` 所有的配置选项，可以使用 `opensm --create-config ./opensm-full.conf` 来生成配置文件模板。

[opensm-full-template](../conf/opensm_full.conf)。

QoS 部分的主要配置摘录与解释如下：

```conf

# qos_*（全局默认，所有端口的兜底默认值）
#     └── qos_ca_*       覆盖 Channel Adapter 的 HCA 端口
#     └── qos_sw0_*      覆盖交换机 port 0（管理端口）
#     └── qos_swe_*      覆盖交换机所有外部端口（external ports）
#         └── qos_sw2sw_*    进一步覆盖其中的 switch-to-switch 端口
#     └── qos_rtr_*      覆盖 Router 端口

# Enable QoS setup
qos TRUE

# QoS policy file to be used
qos_policy_file (null)

# Suppress QoS MAD status errors
suppress_sl2vl_mad_status_errors FALSE

# Override multicast SL provided in join/create request
# OpenSM uses the given SL to override the SL in the request
# 0xff disables the feature
# NOTE: Debug option. Changing the value is not recommended.
override_create_mcg_sl 0xff

# QoS default options
qos_max_vls 0
qos_high_limit -1
qos_vlarb_high (null)
qos_vlarb_low (null)
qos_sl2vl (null)

# QoS CA options
qos_ca_max_vls 0
qos_ca_high_limit -1
qos_ca_vlarb_high (null)
qos_ca_vlarb_low (null)
qos_ca_sl2vl (null)

# QoS Switch Port 0 options
qos_sw0_max_vls 0
qos_sw0_high_limit -1
qos_sw0_vlarb_high (null)
qos_sw0_vlarb_low (null)
qos_sw0_sl2vl (null)

# QoS Switch external ports options
qos_swe_max_vls 0
qos_swe_high_limit -1
qos_swe_vlarb_high (null)
qos_swe_vlarb_low (null)
qos_swe_sl2vl (null)

# QoS Switch-to-switch external port options
qos_sw2sw_max_vls 0
qos_sw2sw_high_limit -1
qos_sw2sw_vlarb_high (null)
qos_sw2sw_vlarb_low (null)
qos_sw2sw_sl2vl (null)

# QoS Router ports options
qos_rtr_max_vls 0
qos_rtr_high_limit -1
qos_rtr_vlarb_high (null)
qos_rtr_vlarb_low (null)
qos_rtr_sl2vl (null)
```

配置文件准备好之后，在启动 OpenSM 的时候使用 `-F` 参数加载配置即可

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -F ./opensm.conf'
```

---

### 16.6.3 需求场景

接下来，我们假设一个需求，然后看看 IB QoS 的解决方案。

设想一个 AI 训练集群的共享 IB 子网，某条链路上同时承载三类性质各异的 RDMA 流量：

- **存储与检查点（Checkpoint）**：训练过程中，框架会周期性地将模型权重写入 Lustre、GPFS 等分布式文件系统，走的是 NFS over RDMA 或 NVMe-oF 协议。这类流量的带宽需求不算最高，但有严格的时间窗口，如果 Checkpoint 写入被长时间阻塞，超出框架的超时阈值，整个训练作业会直接失败。
- **AllReduce 梯度同步**：GPU 之间的梯度聚合，是整条链路的主力带宽消耗者，也是典型的大象流，突发性强，一旦开始就会持续占用大量缓冲。
- **AllGather 及后台流量**：参数广播、模型导出、监控采集等。带宽需求低于 AllReduce，但同样具有突发性，优先级最低，尽力而为即可，但也不能被彻底饿死。

需要特别说明的是，AI 训练框架的**协调与信令**（任务调度、barrier 协商、心跳检测）通常走 TCP/IP 带外网络，并不经过 IB RDMA 数据路径，因此不在我们的 QoS 设计范围之内。

网络团队提出三条具体要求：

1. **隔离**：三类流量必须使用各自独立的缓冲，防止 AllReduce 大象流将 Checkpoint 流量队头阻塞；
2. **优先级**：存储流量优先于训练流量，能够在队列中"插队"，保证 Checkpoint 写入的时延；
3. **带宽分配 + 防饿死**：平时带宽主要让给 AllReduce；存储流量出现时能够抢到带宽；但存储流量不能将 AllReduce 和 AllGather 彻底饿死，必须保证它们周期性地获得服务。

### 16.6.3 把需求翻译成 IB QoS 配置

这三条要求，恰好一一对应到我们前面讲的三个机制：

| 需求          | 用什么机制                           | 怎么配                                                        |
| ------------- | ------------------------------------ | ------------------------------------------------------------- |
| 隔离          | **SL2VL**                            | 把三类流量的 SL 映射到不同的 VL，天然获得独立缓冲             |
| 优先级        | **VLArb 高优先级表**                 | 存储流量的 VL 进高表，训练和后台在低表                        |
| 带宽 + 防饿死 | **VLArb 低优先级权重 + VLHighLimit** | AllReduce 在低表拿大权重；用 VLHighLimit 限制高表每轮最大消耗 |

**SL 到 VL 的划分：**

16 个 SL 分成三段，SL15 按惯例保留映射到 VL15 供 SMA 使用：

| SL 范围  | 流量类型           | 映射到  |
| -------- | ------------------ | ------- |
| SL 0–5   | 存储/Checkpoint    | **VL0** |
| SL 6–10  | AllReduce 梯度同步 | **VL1** |
| SL 11–15 | AllGather / 后台   | **VL2** |

**VLArb 表设计：**

VL0（存储）单独进高优先级表，VL1 和 VL2 在低优先级表中按权重轮转：

- **高优先级表**（`qos_vlarb_high`）：仅 VL0，权重 4。只要高表有数据，调度器优先服务 VL0，确保 Checkpoint 写入不被训练流量抢占。
- **低优先级表**（`qos_vlarb_low`）：VL1 权重 28，VL2 权重 4。AllReduce 拿到低表 87.5% 的带宽份额，后台流量拿到 12.5%。
- **VLHighLimit = 4**：高表每轮最多连续消耗 4 个 credit，累计达到上限后调度器强制切换到低表服务一次，之后高表重置计数。这一机制保证了即便存储流量持续不断，AllReduce 也必然周期性地获得发送机会，不会被饿死。

最终 opensm.conf 配置如下：

```bash
# opensm.conf
# 指定路由引擎为 minhop，和 qos 无关，opensm 默认会试着启用 `ar_updn`
routing_engine minhop

qos TRUE
# 启用 QoS 功能，opensm 才会下发 SL2VL 表和 VLArb 表

max_op_vls          4
# OperVLs 字段的枚举编码，4 对应启用 VL0–VL7（8条VL），
# 足够覆盖我们设计中使用的 VL0、VL1、VL2

qos_max_vls         8
# 子网允许协商的最大 VL 数，直接填数量（非枚举码），
# 与 max_op_vls 4 保持语义一致

qos_sl2vl           0,0,0,0,0,0,1,1,1,1,1,2,2,2,2,2
# 16 个 SL 到 VL 的映射，按顺序对应 SL0–SL15：

qos_vlarb_high      0:4
# 高优先级仲裁表：仅 VL0，权重 4
# 格式为 VL:weight，调度器优先从高表选包发送
# weight 的有效范围是 0–255（uint8_t）

qos_vlarb_low       1:28,2:4
# 低优先级仲裁表：VL1 权重 28，VL2 权重 4
# VL1 占低表带宽的 87.5%，VL2 占 12.5%

qos_high_limit      4
# VLHighLimit 全局默认值：高优先级表每轮最多连续消耗 4 个 credit，
# 达到上限后调度器强制切换到低表服务一次，防止 VL0 饿死 VL1/VL2

```

### 16.6.4 验证一：接口信息

启动 openSM：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -F ./opensm.conf'
-------------------------------------------------
OpenSM 5.21.12.MLNX20250617.f74e01b8
Config file is `./opensm.conf`:
 Reading Cached Option File: ./opensm.conf
 Loading Cached Option:routing_engine = minhop
 Loading Cached Option:max_op_vls = 4
 Loading Cached Option:qos_max_vls = 8
 Loading Cached Option:qos_sl2vl = 0,0,0,0,0,0,1,1,1,1,1,2,2,2,2,2
 Loading Cached Option:qos_vlarb_high = 0:4
 Loading Cached Option:qos_vlarb_low = 1:28,2:4
 Loading Cached Option:qos_high_limit = 4
Command Line Arguments:
 Log File: /var/log/opensm.log
-------------------------------------------------
OpenSM 5.21.12.MLNX20250617.f74e01b8

ibwarn: [20747] sim_connect: attached as client 1 at node "Sw1"
Using default GUID 0x200000
Entering DISCOVERING state

OpenSM $ Entering MASTER state
OpenSM $
```

接着，新开一个 terminal，读一下 PortInfo 的 `OperVLs` 和 `VLHighLimit`：

```bash
# 加载环境变量
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so

# 查询 HcaA port 1 的端口信息
expert@net21:~$ smpquery PortInfo 2 1
ibwarn: [20700] sim_connect: attached as client 1 at node "Sw1"
# Port info: Lid 2 port 1
Mkey:............................<not displayed>
GidPrefix:.......................0xfe80000000000000
Lid:.............................2
SMLid:...........................1
...
VLCap:...........................VL0-7
InitType:........................0x00
VLHighLimit:.....................0                         # VLHighLimit 没有生效，应该改成 4，这是 ibsim 已知的问题
VLArbHighCap:....................8
VLArbLowCap:.....................8
InitReply:.......................0x00
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................0
OperVLs:.........................VL0-7                     # 已经启用了 8 条 VL
...

# 查询 Sw1 port 1 的端口信息
expert@net21:~$ smpquery PortInfo 1 1
ibwarn: [20714] sim_connect: attached as client 1 at node "Sw1"
# Port info: Lid 1 port 1
Mkey:............................<not displayed>
GidPrefix:.......................0x0000000000000000
Lid:.............................0
SMLid:...........................0
...
VLCap:...........................VL0-7
InitType:........................0x00
VLHighLimit:.....................0                         # VLHighLimit 没有生效，应该改成 4，这是 ibsim 已知的问题
VLArbHighCap:....................8
VLArbLowCap:.....................8
InitReply:.......................0x00
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................16
OperVLs:.........................VL0-7                     # 已经启用了 8 条 VL
...
```

> ibsim 当前不支持对 VLHighLimit 的处理。从源码中可以看出来：sim_net.c 第 1269–1307 行，update_portinfo() 是每次 GET/SET 响应前调用的函数，负责把 Port 结构体的字段写回 p->portinfo[]。ibsim 不对 VLHighLimit 字段进行处理，这是模拟器限制。
>
> https://github.com/linux-rdma/ibsim/blob/master/ibsim/sim_net.c#L1269
>
> 因此，这里只能意会一下了，看不到实验结果。

### 16.6.4 验证二：SL2VL 把两类流量分到了不同 VL

应用策略之后，看看 SL2VL table：

```bash
# HcaA 的 SL2VL table：
expert@net21:~$ smpquery SL2VLTable 2
ibwarn: [20792] sim_connect: attached as client 2 at node "Sw1"
# SL2VL table: Lid 2
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|


# Sw1 的 SL2VL table：
expert@net21:~$ smpquery SL2VLTable 1 1
ibwarn: [20794] sim_connect: attached as client 2 at node "Sw1"
# SL2VL table: Lid 1
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  1, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  2, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  3, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  4, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  5, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  6, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  7, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
ports: in  8, out  1: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|
```

映射已经从基线的交替 `0,1,0,1...` 变成了我们设计的 **SL0–5 → VL0、SL6–10 → VL1、SL11–15 → VL2**，和 `qos_sl2vl` 配置一字不差。

### 16.6.5 验证三：VLArb 表

应用策略之后，看看各接口的仲裁表：

```bash
# HcaA port 1 的仲裁表
expert@net21:~$ smpquery VLArbitration 2 1
ibwarn: [20797] sim_connect: attached as client 2 at node "Sw1"
# VLArbitration tables: Lid 2 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x1 |0x2 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x1C|0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |

# Sw1 port 1 的仲裁表
expert@net21:~$ smpquery VLArbitration 1 1
ibwarn: [20799] sim_connect: attached as client 2 at node "Sw1"
# VLArbitration tables: Lid 1 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x1 |0x2 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x1C|0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
```

输出是十六进制，换算后逐项对照我们的配置，全部生效：

| Table         | Hex                 | Dec                   | Qos Conf                 |
| ------------- | ------------------- | --------------------- | ------------------------ |
| High priority | `VL0:0x4`           | VL0=**4**             | `qos_vlarb_high 0:4`     |
| Low priority  | `VL1:0x1C, VL2:0x4` | VL1=**28**, VL2=**4** | `qos_vlarb_low 1:28,2:4` |

---

## 16.7 真实环境下的测试

由于 ibsim 没有数据平面，因此这里我们只能在 openSM 的控制平面对配置进行验证。

现实中，如果有 IB 硬件想要真正观察 QoS 的效果，可以用 `ib_send_bw` 之类工具在不同 SL 上同时打流（`--sl` 参数指定 SL，默认是 0），再用端口计数器 `perfquery` 观察各 VL 实际承载的字节数。这部分留待有硬件时验证。

---

## 16.8 SL 的分配和管理问题

写到这里，讲完了 IB QoS 的队列以及调度机制。但是，熟悉以太网的朋友们应该还会有一个疑问：IB 的 QoS marking 怎么做？

我想先说明一点：InfiniBand 的 QoS 机制在架构上有一个根本性的前提假设：数据包头里的 SL 字段由**发送方**在建立 QP 时填入，交换机在转发过程中不会修改它，只会依据它做调度决策。这意味着整套 QoS 体系的有效性，从一开始就建立在对 ULP 合规行为的信任之上：应用完全可以无视任何策略，在建 QP 时任意填写一个 SL 值。

这与以太网的 QoS 机制形成了本质差异。以太网交换机可以在入端口强制改写数据包的 DSCP 或 802.1p 标记，策略执行点在网络设备上，应用难以绕过。而在 IB 网络里，"marking"发生在连接建立之前的 PathRecord 查询阶段，SM 通过返回携带特定 SL 的 PathRecord 来"建议"应用使用某个服务级别，但这个建议能否被遵守，完全取决于 ULP 的行为。两者的对比如下表所示：

|                  | 以太网                       | InfiniBand                    |
| ---------------- | ---------------------------- | ----------------------------- |
| Marking 发生在   | 端口入方向（可强制 rewrite） | PathRecord 返回时（建连前）   |
| 数据面能改标记吗 | 能（交换机 rewrite DSCP）    | 不能（IB 交换机不改 SL）      |
| 应用能绕过吗     | 难（交换机强制执行）         | 能（无视 PathRecord 里的 SL） |
| 正常工作依赖     | 交换机策略                   | ULP 合规行为                  |

在实际部署中，SL 的分配通常通过两种路径实现。

第一种是由应用或框架层**直接硬编码或通过环境变量指定**。以 AI 训练场景中广泛使用的 NCCL 为例，它在初始化阶段需要快速建立大量 RC QP，为了避免逐一查询 SA 带来的延迟，NCCL 直接在建 QP 时填入 SL 值，默认为 0，并通过环境变量 `NCCL_IB_SL` 向运维人员开放覆盖入口。这种方式简单直接，但 SL 的分配策略必须在集群层面通过运维约定来保证一致性。

第二种是依赖 **SM 的高级 QoS 策略**。OpenSM 支持在 `qos-policy.conf` 中定义一套匹配规则：将端口按 GUID、节点类型或所属 Partition 归组，再按流量特征（源/目端口组、PKey、Service ID）将不同类型的 PathRecord 查询映射到预定义的 SL。走这条路径的 ULP（如 MPI、SRP、iSER）在发起路径查询时不需要感知 SL 的存在，SM 在返回 PathRecord 时已经透明地完成了流量分类和标记。

两种方式在大型 AI 集群中往往并存：NCCL 训练流量通过环境变量约定走特定 SL，存储和管理流量则通过 SM 策略自动完成分类，各自管理彼此独立的那部分流量。

受限于本章的篇幅，我们暂且就不对“高级 QoS 策略”进行展开了。

---

## 16.9 小结

**SL 与 VL 是 IB QoS 的一体两面**：SL 是贴在包上、端到端不变的"等级标签"（4 位，LRH 内）；VL 是链路上拥有独立缓冲和独立信用的"物理队列"（VL0–14 数据 + VL15 管理）。SL 表达意图，VL 落实资源，二者刻意解耦，以适应逐跳变化的链路。

**两张表把它们粘合起来**：SL2VL 映射表回答"这个 SL 该进哪条 VL"（在交换机上按入/出端口细分，SL 不变而 VL 可逐跳改变）；VLArb 仲裁表则负责调度——高/低优先级表 + 权重 + VLHighLimit，即"严格优先级 + 加权轮询 + 防饿死"。这套机制，配合 IB 的信用流控，让 IB 在 AI 训练与高性能计算场景中发挥自如；以太网目前只能说在尽力逼近、模拟它，但在不改变"尽力而为（best-effort）"这条基因的前提下，面对大型 AI 训练，仍很难做到和 IB 同一水平，真正想做到 1:1 替换，恐怕得等 Ultra Ethernet 完全落地。

最后，**别忘了 SL 是谁填的**：IB 的 QoS 有一个隐含前提：SL 由发送方（ULP）在建 QP 时自己填，交换机一路只读不改。所以整套 QoS 的有效性，建立在 ULP 合规之上：SM 通过 PathRecord 返回的 SL 只是"建议"，应用完全可以无视。实践中 SL 的分配走两条路：要么应用层直接指定，要么交给 SM 的 `qos-policy.conf` 按流量特征自动分类、透明地填进 PathRecord。**换句话说，IB 把"队列与调度"做到了极致的集中与确定，却把"谁该用哪个 SL"这第一步，交还给了端侧的信任。**

至此，我们已经能在一张 IB fabric 上既"划分租户、隔离流量"，又"给流量分级、按级调度"了。
