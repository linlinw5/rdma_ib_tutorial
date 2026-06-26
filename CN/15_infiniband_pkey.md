# 第十五章：Infiniband Fabric 的分区（Partition）

## 15.1 从一个熟悉的问题说起

到目前为止，我们搭出来的 IB 子网都有一个共同特点：**所有节点彼此可达，谁都能给谁发包。** 子网管理器算好 LID、填好转发表，整张 fabric 就是一个扁平的、全连通的二层网络。

不过这里要特别提醒一句，**"彼此可达"并不等于 IB 里存在一个像以太网那样的"全网广播"机制**。 众所周知，以太网的地址发现（ARP）靠的是二层广播：不知道对端 MAC，就在广播域里吼一嗓子"谁是这个 IP"，等对方应答。IB 没有这种广播式的地址发现。

回顾第二章讲过的内容：IB 通信的发起，需要发起方事先持有对端的某种地址标识：通常是 IP 地址或 GID，最终要折算成 LID 才能寻址。实际工程中，这个 LID 的来源大体有三种：

- 靠带外 TCP 通道提前把对端参数（含 LID）通告过来（NCCL 等 AI 训练框架普遍采用的方式）；
- 靠 RDMA CM（rdma_resolve_addr + rdma_resolve_route）在建连之前经由 SA PathRecord 查询换得；
- 或者在测试和 HPC 场景里直接静态写入。

换句话说，IB 子网内的"全连通"指的是"只要你已经持有对端的 LID 或 GID，SA 就一定能返回完整的 path record，转发表也一定能把包送到"，而不是"你可以广播去发现网络里的任何人"。

> 严格来说，IB 并非完全没有"一对多"能力，它有由 SM 显式管理的多播（multicast，MGID/MLID），IPoIB 正是借一个多播组来模拟以太网的广播语义，ibacm 也用 IB 多播来做类 ARP 的地址解析。但这些都是需要 SM 分配、应用显式加入的机制，完全依赖集中管理，和以太网那种"默认人人都收得到"的隐式广播是两回事；在主流 AI/HPC 工作负载中几乎见不到这种用法。

**真实的数据中心对网络的需求往往不会到“互联互通”就停止，它往往还有“分区、控制”方面的需求**。例如，同一套物理网络上往往跑着多个互不信任的租户、多套互不相干的业务：A 部门的训练集群不该看到 B 部门的存储流量，一个 GPU 租户的 AllReduce 不该误打误撞地发到另一个租户的节点上。

在以太网世界里，解决这个问题的工具我们再熟悉不过：**VLAN**。一根物理网线、一台物理交换机，通过给端口打上不同的 VLAN ID，就能把一张物理网络切成若干个互相隔离的广播域，谁和谁能在二层上通信，由 VLAN 划分说了算。

InfiniBand 面对的是同样的需求，也给出了一个**功能上高度类似、但机制上完全不同**的答案：**分区（Partition）**，其载体是一个叫 **P_Key（Partition Key，分区键）** 的 16 位标识。

不过，用 VLAN 来打这个比方，有一点必须先说在前头，否则容易把方向带偏：**VLAN 设计的初衷是隔离广播域。** 以太网广播太多，需要把一个大二层切成若干小块，抑制广播风暴、缩小故障域，所以隔离“谁和谁能通信”更像是切分广播域之后顺带得到的结果。而 IB 本就没有广播，partition 的出发点从一开始就**不是"抑制广播"，而是"控制谁能和谁通信"**。

所以单从目的上讲，IB 的 partition 其实更接近**访问控制（ACL）**；如果你有存储网络的背景，会发现它更像 **SAN 里的 Zoning**：在一张共享的物理 fabric 上，明确规定哪些端口被允许互相看见、互相通信，默认互不可见。所以，**VLAN 是"切广播域"，partition 是"定可达关系"**。记住这个出发点上的差异，后面成员位、默认分区那些设计为什么是现在这个样子，就会顺理成章。

---

## 15.2 P_Key：分区的载体

先把最核心的对象立起来。

**P_Key 是一个 16 位的值，每个 IB 端口都持有一张 P_Key 表（P_Key Table），记录着"我属于哪些分区"。** 一个端口可以同时属于多个分区，就像一个以太网 trunk 口可以承载多个 VLAN。

这 16 位并不是平铺的一个数字，它被拆成两部分：

```
bit15        bit14 ........................ bit0
┌──────────┐ ┌──────────────────────────────────┐
│ Member   │ │     15-bit partition key value   │
│ (1 bit)  │ │          (P_Key value)           │
└──────────┘ └──────────────────────────────────┘
```

- **低 15 位**：才是真正用来区分"哪个分区"的键值。两个端口要想通信，这 15 位必须相等，这相当于 VLAN ID 要一致。
- **最高位（bit 15）**：是**成员类型位（membership bit）**，它不参与"是不是同一个分区"的判断，而是标记"我在这个分区里是什么身份"：
  - `1` = **完全成员（Full member）**
  - `0` = **受限成员（Limited member）**

这个成员位的具体作用，下一节专门介绍。这里先记住一个换算：同一个分区键 `0x0100`，完全成员在 P_Key 表里存成 `0x8100`（最高位置 1），受限成员存成 `0x0100`。

> 我们在第十章介绍 BTH（Base Transport Header）时，重点看了 DLID/SLID、QPN、PSN 这几个字段。其实 BTH 里还有一个当时没展开的 16 位字段，就是 **P_Key**。**每一个 IB 数据包，都在 BTH 里携带着它所属分区的 P_Key。** 这就是分区隔离能够在每个包上生效的物理基础：接收方拿到包，就能从 BTH 里读出它来自哪个分区，再和自己的 P_Key 表比对，决定收还是丢。

---

## 15.3 Full 与 Limited：一条不对称的隔离规则

如果 IB 的分区只是"键值相等才能通信"，那它和 VLAN 几乎没有区别。但成员位的引入，让 IB 多出了一层新能力：**同一个分区内部，还能再分出"能互相通信的"和"不能互相通信的"两类成员。**

规则只有一句话，但要看仔细：

> **两个端口能通信，当且仅当：它们 P_Key 的低 15 位相等，且二者之中至少有一个是完全成员。**

把它展开成一张表：

| 发起方 \ 对端           | 完全成员（Full） | 受限成员（Limited） |
| ----------------------- | ---------------- | ------------------- |
| **完全成员（Full）**    | ✅ 可通信        | ✅ 可通信           |
| **受限成员（Limited）** | ✅ 可通信        | ❌ **不可通信**     |

换句话说：

- **完全成员**之间随便通信；
- **完全成员 ↔ 受限成员**可以通信；
- **受限成员 ↔ 受限成员**——即使在同一个分区里、键值完全相同——也**不能**通信。

> 呵呵，做网络的朋友们应该立刻能意识到，这算啥“新能力”，不就是以太网的 `Private VLAN` 嘛，的确如此。

这条规则的实际用途非常典型：**做"多对一"的星型隔离。** 比如一套共享存储，存储节点设为完全成员，所有客户端设为受限成员。结果就是：每个客户端都能访问存储（受限 ↔ 完全，通），但客户端之间彼此看不见（受限 ↔ 受限，不通）。你不需要为每一对"该隔离的客户端"单独划分区，一个分区 + 成员位就搞定了。

---

## 15.4 默认分区与管理通道：为什么"划了区还能被 SM 管"

讲到这里，也许你会有一个问题：**如果我把所有端口都划进各自的租户分区，子网管理器（SM）还怎么和它们通信？SM 自己又属于哪个分区？**

IB 对此有一套兜底设计。

**默认分区（Default Partition）。** 存在一个保留的分区键 `0x7FFF`，称为默认分区。在没有任何分区配置时，所有端口都是默认分区的完全成员。这就是为什么前面我们从没配过分区，整张网却全连通的原因：大家其实都待在同一个默认分区里。

- `0xFFFF`：默认分区的**完全成员**
- `0x7FFF`：默认分区的**受限成员**

**管理流量走默认分区。** SM、SA 这些管理实体，正是通过默认分区与所有节点通信的。所以默认分区不能被破坏，否则 SM 一旦把某个节点划出去，就再也管不到它，子网直接失联。

OpenSM 对此有一个关键的**保护行为**：当你定义了租户分区、把某个端口从默认分区的完全成员里拿掉时，OpenSM 仍然会**自动给它保留默认分区的受限成员资格（`0x7FFF`）**。这样一来：

- 该端口和 SM（默认分区完全成员）之间：受限 ↔ 完全，**通**——管理通道保住了；
- 该端口和别的租户端口之间（如果对方也只是默认分区受限成员）：受限 ↔ 受限，**不通**——租户隔离也成立了。

成员位的不对称规则，在这里被 OpenSM 用得淋漓尽致：**它让管理平面始终可达，同时不破坏租户之间的数据平面隔离。**

---

## 15.5 P_Key 表落到哪里：端口、交换机与"执行"

分区配置最终要变成每个端口上的一张 P_Key 表，这一步由 SM 在子网初始化时完成，机制和前几章我们看 LFT 下发是一样的：**SM 算好每个端口该有哪些 P_Key，通过 SMP（子网管理包）的 Set 操作写进去（PKeyTable 属性）。**

但"持有 P_Key 表"和"真正执行隔离"是两件事，这里要分清楚：

- **HCA 端口（终端节点）**：**Mandatory 强制** P_Key 检查。网卡发包时给 BTH 填上 P_Key，收包时拿包里的 P_Key 和自己表里的比对，不匹配就丢弃，并可向 SM 上报一个"P_Key 违例"的 trap。这是强制的，关不掉。
- **交换机外部端口**：P_Key 的执行是 **Optional**（PartitionEnforcementInbound / PartitionEnforcementOutbound）。如果交换机支持并开启，它会在转发时就地做 P_Key 过滤，把不合规的包在网络中途就拦下来，不让它白白占用到达目的地的带宽；如果不支持，则隔离完全交给两端的 HCA 兜底。
  - Inbound：从这个端口进入交换机的包做 P_Key 检查。也就是说，从外部 HCA 发过来、刚到达这个端口的流量，在被转发之前先查一遍 P_Key。
  - Outbound：从这个端口离开交换机的包做 P_Key 检查。也就是说，交换机已经决定要从这个端口转发出去的流量，在真正发出去之前再查一遍 P_Key。

这个"终端强制、交换机可选"的分工，和以太网里"VLAN 过滤主要在交换机端口上做"的直觉正好相反，值得留意：**IB 的分区隔离，最终的、保底的执行者是终端网卡，而不是交换机。**

至于一张 P_Key 表能放多少个分区，由端口的 PartitionCap 属性决定（HCA 常见为 128），表项以 32 个为一块（block）组织。这些是实现细节，知道有这么回事即可。

---

## 15.6 实验测试

首先，先给出openSM的帮助文档地址，在这里可以找到 partitions.conf 的相关格式说明：

https://manpages.debian.org/testing/opensm/opensm.8.en.html

其次，opensm 启动时，与 partition 相关的参数：

```bash
# -P 用于指定 partitions.conf 配置文件地址
--Pconfig, -P <partition-config-file>
          This option defines the optional partition configuration file.
          The default name is '/etc/opensm/partitions.conf'.

# -Z 用于指定交换机的 enforcement 策略，对于 ibsim 来说是无效的
--part_enforce, -Z [both, in, out, off]
          This option indicates the partition enforcement type (for switches)
          Enforcement type can be outbound only (out), inbound only (in), both or
          disabled (off). Default is both.
```

我们为接下来的实验专门设计一个 ibsim 拓扑：

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

### 15.6.1 测试一：观察默认的 partition table

启动模拟器并 `dump`：

```bash
expert@net21:~$ rlwrap ibsim -s ./ib4.net

sim> dump
# Net status - Mon Jun 22 08:24:19 2026

Switch 8 "Sw1"  nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 0 portchange 1
200000  [0]     "Sma Port"[0]    lid 0 lmc 0 smlid 0  4x  2.5G Active/LinkUp
200000  [1]     "HcaA"[1]         4x  2.5G Init/LinkUp
200000  [2]     "HcaB"[1]         4x  2.5G Init/LinkUp
200000  [3]     "HcaC"[1]         4x  2.5G Init/LinkUp
200000  [4]     "HcaD"[1]         4x  2.5G Init/LinkUp
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Ca 2 "HcaA"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Sw1"[1]         lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaB"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Sw1"[2]         lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaC"     nodeguid 100006 sysimgguid 100006
100007  [1]     "Sw1"[3]         lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100008  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaD"     nodeguid 100009 sysimgguid 100009
10000a  [1]     "Sw1"[4]         lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
10000b  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 5 nodes
```

记录各 Hca 的 GUID 如下：

| Node | Port GUID |
| ---- | --------- |
| HcaA | 0x100001  |
| HcaB | 0x100004  |
| HcaC | 0x100007  |
| HcaD | 0x10000a  |

启动 openSM，暂时不加载任何 partition 配置：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -f -'
```

观察ibsim，LID 列表如下：

| Node | LID |
| ---- | --- |
| Sw1  | 1   |
| HcaA | 2   |
| HcaB | 5   |
| HcaC | 8   |
| HcaD | 9   |

用 `smpquery pkeys` 读取每个端口实际拿到的 P_Key 表：

```bash
expert@net21:~$ smpquery PKeyTable 1
ibwarn: [14139] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
8 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 2
ibwarn: [14141] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 5
ibwarn: [14143] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 8
ibwarn: [14145] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 9
ibwarn: [14147] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
```

可以看到，对于 ibsim 模拟出来的设备，交换机默认有 8 个 pkeys capacity，而 Hca 则默认有 64 个 pkeys capacity。

当前所有的设备都自动分配了一个默认的 partition key：0xffff，代表大家都是“默认分区的完全成员”。

相当于 partitions.conf 配置为：

```conf
# p1.conf
#
# keywords for PortGUID definition:
# - 'ALL' means all end ports in this subnet.
# - 'ALL_CAS' means all Channel Adapter end ports in this subnet.
# - 'ALL_SWITCHES' means all Switch end ports in this subnet.
# - 'ALL_ROUTERS' means all Router end ports in this subnet.
# - 'SELF' means subnet manager's port.

# 所有 HCA 都在默认分区，full 成员
Default=0x7fff : ALL=full ;
```

根据上面的解释，如果想将所有的 Hca 设置为默认分区的受限成员，可以这样写：

```conf
# p2.conf

Default=0x7fff : ALL=limited, SELF=full ;
```

### 15.6.2 测试二：把一张 fabric 切成两个租户

根据上一节 dump 出来的 Hca port GUID，写一个新的 partitions 配置：

```conf
# p3.conf

# 分区 A：HcaA + HcaB
PartA=0x0010 : 0x100001=full, 0x100004=full ;

# 分区 B：HcaC + HcaD
PartB=0x0020 : 0x100007=full, 0x10000a=full ;

# 默认分区只给 SM 自己，不加其他节点
Default=0x7fff : SELF=full ;
```

启动 OpenSM（加载自定义的分区文件）：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -P p3.conf -f -'
```

用 `smpquery pkeys` 读取每个端口实际拿到的 P_Key 表：

```bash
expert@net21:~$ smpquery PKeyTable 1
ibwarn: [14336] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
8 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 2
ibwarn: [14338] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x8010 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 5
ibwarn: [14340] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x8010 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 8
ibwarn: [14342] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x8020 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port

expert@net21:~$ smpquery PKeyTable 9
ibwarn: [14344] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x8020 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
```

从以上输出中，不难发现，每个 Hca 除了拿到了 `0x7fff`(默认分区受限成员) ，还拿到了各自所属区域的完全成员(`0x8010` 或 `0x8020`)。

### 15.6.3 测试三：full vs limited 成员区别

设想一个共享存储的模型：HcaA 当存储节点（完全成员），HcaB / HcaC 充当两个互不信任的客户端（受限成员）， HcaD 不允许访问存储。
改写 `partitions.conf`：

```conf
# p4.conf

# HcaA 是 full member，HcaB/HcaC 是 limited member，HcaD 不在此分区
# limited 之间互相不通，但都能和 HcaA 通信
PartC=0x0030 : 0x100001=full, 0x100004=limited, 0x100007=limited ;

# 默认分区：给 SM 自己分full
Default=0x7fff : SELF=full ;
```

启动 OpenSM（加载自定义的分区文件）：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -P p4.conf -f -'
```

用 `smpquery pkeys` 读取每个端口实际拿到的 P_Key 表：

```bash
expert@net21:~$ smpquery PKeyTable 1
ibwarn: [14394] sim_connect: attached as client 1 at node "Sw1"
   0: 0xffff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
8 pkeys capacity for this port
expert@net21:~$ smpquery PKeyTable 2
ibwarn: [14396] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x8030 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
expert@net21:~$ smpquery PKeyTable 5
ibwarn: [14398] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x0030 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
expert@net21:~$ smpquery PKeyTable 8
ibwarn: [14400] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x0030 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
expert@net21:~$ smpquery PKeyTable 9
ibwarn: [14402] sim_connect: attached as client 1 at node "Sw1"
   0: 0x7fff 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
   8: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  16: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  24: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  32: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  40: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  48: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
  56: 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000 0x0000
64 pkeys capacity for this port
```

分区表总结如下：

| 端口             | P_Key 表内容      | 默认分区中的成员身份 | PartC 分区中的成员身份 |
| ---------------- | ----------------- | -------------------- | ---------------------- |
| HcaA（存储）     | `0x7fff` `0x8030` | 受限成员             | 完全成员               |
| HcaB（客户端）   | `0x7fff` `0x0030` | 受限成员             | 受限成员               |
| HcaC（客户端）   | `0x7fff` `0x0030` | 受限成员             | 受限成员               |
| HcaD（非客户端） | `0x7fff`          | 受限成员             | 无                     |

---

## 15.8 与以太网 VLAN 的对照

最后，我们把 IB 分区和最熟悉的 VLAN 并排放一起，看清它们神似而形不同。

| 维度               | 以太网 VLAN                      | InfiniBand 分区（P_Key）                        |
| ------------------ | -------------------------------- | ----------------------------------------------- |
| **设计初衷**       | 隔离广播域（抑制广播、缩故障域） | **控制可达关系**（更接近 ACL / SAN Zoning）     |
| **标识**           | 12 位 VLAN ID（4094 个可用）     | 16 位 P_Key（15 位键值 + 1 位成员位）           |
| **承载位置**       | 802.1Q 标签（链路层，帧头）      | P_Key 字段（**传输层**，BTH 内）                |
| **谁来划分**       | 网管在交换机端口上手工/协议配置  | **子网管理器集中计算下发**（`partitions.conf`） |
| **隔离粒度**       | 同 VLAN 内二层全连通             | 同分区内还能用成员位再切一刀（完全 / 受限）     |
| **执行者**         | 主要在交换机端口                 | **终端 HCA 强制执行**，交换机可选               |
| **配置范式**       | 分布式：逐台设备配置             | 集中式：一份策略文件，SM 统一推送               |
| **"全连通"默认态** | 默认 VLAN 1                      | 默认分区 `0x7fff` / `0xffff`                    |

最值得网络工程师关注的两点差异：

1. **承载层次不同。** VLAN 标签在链路层的帧头，靠交换机识别；P_Key 在传输层的 BTH 里，最终靠终端网卡识别。

2. **管理范式不同。** VLAN 是分布式配置的产物，你要在每台交换机上各配一遍、还要操心 trunk 和 VLAN 透传；IB 分区是集中式的，只需要写一份 `partitions.conf`，剩下的"算哪个口该有哪些 P_Key、怎么下发"全由 SM 包办。

---

## 15.9 小结

**P_Key 是 IB 的分区载体**，16 位里低 15 位是分区键值（决定"是不是同一个分区"），最高位是成员位（决定"在分区里是完全还是受限身份"）。每个端口持有一张 P_Key 表，记录自己属于哪些分区；每个数据包在 BTH 里携带 P_Key，接收方据此比对、决定收丢。

**默认分区（`0x7fff` / `0xffff`）是管理平面的兜底**：OpenSM 会为被划入租户分区的端口自动保留默认分区的受限成员资格，让 SM 始终可达，同时不破坏租户间的数据隔离。

至此，我们已经能在一张 IB fabric 上划分租户、隔离流量。
