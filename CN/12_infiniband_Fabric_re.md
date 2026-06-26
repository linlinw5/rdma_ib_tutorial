# 第十二章：Infiniband Fabric 路由引擎

## 12.1 先澄清一个容易误解的名字

这一章要讲"路由引擎"（routing engine），但在开始之前，必须先纠正一个望文生义的误会。

这里说的"路由"，**不是** IB 子网之间的路由（那是网络层、靠 IB 路由器和 GID 完成的跨子网转发）。路由引擎是 SM 的一个组件，它干的是另一件事：**在一个 IB 子网内部，计算每台交换机的转发表（LFT）该怎么填。**所以，这里的“路由”是指交换机按转发表转发，在传统的以太网 + TCP/IP 中，我们会说这是“交换”，而非“路由”。但是因为 IB 的寻址涉及多种算法，所以业界在这里普遍接受“路由”的说法。

前一章我们看到 SM 为每台交换机算出了 LFT，用来决定"发往某个 LID 的包从哪个端口出去"。而"怎么算这张表"并不是唯一确定的，这就牵涉到算法问题了：从源到目的往往有多条路可走，先走哪条、如何在多条等价路径间选择、要不要为了避免某些隐患而牺牲最短路。这些决策逻辑，就是 IB 的路由引擎。

所以更准确的理解是：**路由引擎是"子网内转发表的计算策略"。** 同一张物理拓扑，喂给不同的路由引擎，会算出不同的 LFT。OpenSM 内置了多种路由引擎（minhop、updn、ftree、torus-2QoS 等），通过启动参数 `-R <engine>` 选择。

这一章我们聚焦最基础、也最能说明问题的两种：min-hop 和 up-down，用一个精心设计的环形拓扑，看清它们的差别和各自的取舍。

## 12.2 实验拓扑：一个四交换机的环

我们用一个四台交换机首尾相连成环、每台各挂一个 HCA 的拓扑：

```
        SwA ════════ SwB
         ║            ║
         ║            ║
        SwD ════════ SwC
```

ibsim 拓扑文件如下：

```bash
# ib2.net — 四交换机环形拓扑，用于演示 min-hop vs UpDn
#
#        SwA ──────── SwB
#         │            │
#         │            │
#        SwD ──────── SwC
#
# 每台交换机挂一个 HCA。环上任意两点都有顺/逆时针两条路径。

Switch  8 "SwA"
[1]     "HcaA"[1]
[2]     "SwB"[3]
[3]     "SwD"[2]

Switch  8 "SwB"
[1]     "HcaB"[1]
[2]     "SwC"[3]
[3]     "SwA"[2]

Switch  8 "SwC"
[1]     "HcaC"[1]
[2]     "SwD"[3]
[3]     "SwB"[2]

Switch  8 "SwD"
[1]     "HcaD"[1]
[2]     "SwA"[3]
[3]     "SwC"[2]

Hca     2 "HcaA"
[1]     "SwA"[1]

Hca     2 "HcaB"
[1]     "SwB"[1]

Hca     2 "HcaC"
[1]     "SwC"[1]

Hca     2 "HcaD"
[1]     "SwD"[1]
```

为什么偏偏选环形？因为**环是最能暴露 credit loop 风险的结构**。环上任意两个对角节点之间都有两条等长路径（顺时针、逆时针），路由引擎必须做选择；而一旦选择不当，多股流量的缓冲依赖就可能沿环首尾相接、形成死锁。这恰好是检验路由引擎优劣的试金石。

## 12.3 min-hop：只认跳数

第一种路由引擎是 min-hop，思路最朴素：**对每一个目的地，独立地选择跳数最少的路径。** 哪条路最短走哪条，仅此而已。

Min-hop 的问题在于不会考虑全局：它不会去想"我给这么多目的地各自选的方向，叠加起来会不会出问题"。

启动模拟器加载这个拓扑：

```bash
rlwrap ibsim -s ./ib2.net
```

用 `-R minhop` 启动 OpenSM：

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -R minhop -f -'
```

用 `dump` 看初始拓扑和各节点的 LID 分配：

```bash
expert@net21:~$ rlwrap ibsim -s ./ib2.net
parsing: ./ib2.net
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566649] parse_port_connection_data: cannot parse remote lid and connection type
./ib2.net: parsed 40 lines
########################
Network simulator ready.
MaxNetNodes    = 2048
MaxNetSwitches = 256
MaxNetPorts    = 13312
MaxLinearCap   = 49152
MaxMcastCap    = 1024

sim> dump
# Net status - Thu Jun 18 04:34:53 2026

Switch 8 "SwA"  nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]2 [7]3 [8]3 [9]3
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "HcaA"[1]         4x  2.5G Active/LinkUp
200000  [2]     "SwB"[3]          4x  2.5G Active/LinkUp
200000  [3]     "SwD"[2]          4x  2.5G Active/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "SwB"  nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]2 [8]2 [9]3
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "HcaB"[1]         4x  2.5G Active/LinkUp
200001  [2]     "SwC"[3]          4x  2.5G Active/LinkUp
200001  [3]     "SwA"[2]          4x  2.5G Active/LinkUp
200001  [4]                       4x  2.5G Down/Polling
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Switch 8 "SwC"  nodeguid 200002 sysimgguid 200002
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
200002  [0]     "Sma Port"[0]    lid 6 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200002  [1]     "HcaC"[1]         4x  2.5G Active/LinkUp
200002  [2]     "SwD"[3]          4x  2.5G Active/LinkUp
200002  [3]     "SwB"[2]          4x  2.5G Active/LinkUp
200002  [4]                       4x  2.5G Down/Polling
200002  [5]                       4x  2.5G Down/Polling
200002  [6]                       4x  2.5G Down/Polling
200002  [7]                       4x  2.5G Down/Polling
200002  [8]                       4x  2.5G Down/Polling

Switch 8 "SwD"  nodeguid 200003 sysimgguid 200003
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]3 [6]3 [7]0 [8]3 [9]1
200003  [0]     "Sma Port"[0]    lid 7 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200003  [1]     "HcaD"[1]         4x  2.5G Active/LinkUp
200003  [2]     "SwA"[3]          4x  2.5G Active/LinkUp
200003  [3]     "SwC"[2]          4x  2.5G Active/LinkUp
200003  [4]                       4x  2.5G Down/Polling
200003  [5]                       4x  2.5G Down/Polling
200003  [6]                       4x  2.5G Down/Polling
200003  [7]                       4x  2.5G Down/Polling
200003  [8]                       4x  2.5G Down/Polling

Ca 2 "HcaA"     nodeguid 100000 sysimgguid 100000
100001  [1]     "SwA"[1]         lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaB"     nodeguid 100003 sysimgguid 100003
100004  [1]     "SwB"[1]         lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaC"     nodeguid 100006 sysimgguid 100006
100007  [1]     "SwC"[1]         lid 8 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100008  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaD"     nodeguid 100009 sysimgguid 100009
10000a  [1]     "SwD"[1]         lid 9 lmc 0 smlid 1  4x  2.5G Active/LinkUp
10000b  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 8 nodes

```

为后面分析方便，先记下各节点的 LID 归属：

| 节点 | LID |
| ---- | --- |
| SwA  | 1   |
| HcaA | 2   |
| SwB  | 3   |
| HcaB | 5   |
| SwC  | 6   |
| SwD  | 7   |
| HcaC | 8   |
| HcaD | 9   |

同时，我们也把四台交换机的转发表总结如下：

```bash
"SwA"  nodeguid 200000
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]2 [7]3 [8]3 [9]3
[0]     "Sma Port"[0]    lid 1
[1]     "HcaA"[1]
[2]     "SwB"[3]
[3]     "SwD"[2]

"SwB"  nodeguid 200001
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]2 [8]2 [9]3
[0]     "Sma Port"[0]    lid 3
[1]     "HcaB"[1]
[2]     "SwC"[3]
[3]     "SwA"[2]

"SwC"  nodeguid 200002
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
[0]     "Sma Port"[0]    lid 6
[1]     "HcaC"[1]
[2]     "SwD"[3]
[3]     "SwB"[2]

"SwD"  nodeguid 200003
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]3 [6]3 [7]0 [8]3 [9]1
[0]     "Sma Port"[0]    lid 7
[1]     "HcaD"[1]
[2]     "SwA"[3]
[3]     "SwC"[2]
```

这套转发表里，每台交换机对每个目的 LID 都选了最短路。表面上看一切合理——直到我们把这些选择叠加起来看全局。

SM 启动日志的主线和前一章一样（DISCOVERING → HEAVY SWEEP → MASTER → SUBNET UP），其中能看到路由引擎的标记 "minhop tables configured on all switches"：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -R minhop -f -'
...
******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 18 04:34:30 730600 [EFC006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 18 04:34:30 755027 [EFC006C0] 0x80 -> Entering MASTER state
Jun 18 04:34:30 769755 [EFC006C0] 0x02 -> osm_ucast_mgr_process: minhop tables configured on all switches
Jun 18 04:34:30 811513 [EFC006C0] 0x02 -> SUBNET UP
```

## 12.4 min-hop 埋下的隐患：一个真实的 credit loop

我们要用上面这套真实的 min-hop 转发表，还原出一个潜在的死锁。

### credit loop 到底是什么

先纠正一个直觉误区：credit loop **不是**"包在路由上绕圈、无限循环"那种环。它是**缓冲区的循环等待**。

回忆 IB 的无损流控：交换机要把一个包发往下一跳，必须等下一跳给出"credit"，也就是确认它有空闲缓冲区能接收。没有credit，包就在原地缓冲区里等着。

现在设想若干股流量，它们占用的链路和等待的链路首尾相接、绕成一圈：第一股占着某条链路、等第二股释放；第二股占着下一条、等第三股；......最后一股回过头来等第一股。每一股都在等下一股先动，而没有任何一股能先动，结果就是所有缓冲区永久卡死。这就是 credit loop，本质和操作系统里"A 等 B 的锁、B 等 A 的锁"的死锁完全一样，只不过这里等的是缓冲信用。

要点是：**单独一条路径永远不会死锁，credit loop 必须是多股流量的占用方向叠加成环。**

### 从 min-hop 转发表里把这个环找出来

为了简化示例，在这个例子中，我们暂且只关注交换机之间的流量。环的物理结构是四条链路：A–B、B–C、C–D、D–A。

```bash
#        SwA ──────── SwB
#         │            │
#         │            │
#        SwD ──────── SwC
```

从 min-hop 的转发表里，提取每台交换机转发到"对角交换机"的方向（这些是要走两跳、需要选方向的流量）：

```bash
"SwA"  nodeguid 200000
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]2 [7]3 [8]3 [9]3
[0]     "Sma Port"[0]    lid 1
[1]     "HcaA"[1]
[2]     "SwB"[3]
[3]     "SwD"[2]

"SwB"  nodeguid 200001
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]2 [8]2 [9]3
[0]     "Sma Port"[0]    lid 3
[1]     "HcaB"[1]
[2]     "SwC"[3]
[3]     "SwA"[2]

"SwC"  nodeguid 200002
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
[0]     "Sma Port"[0]    lid 6
[1]     "HcaC"[1]
[2]     "SwD"[3]
[3]     "SwB"[2]

"SwD"  nodeguid 200003
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]3 [6]3 [7]0 [8]3 [9]1
[0]     "Sma Port"[0]    lid 7
[1]     "HcaD"[1]
[2]     "SwA"[3]
[3]     "SwC"[2]

# 从表中可以看到
 SwA 去 SwC(lid6) → 出端口 2 → 方向 A→B
 SwB 去 SwD(lid7) → 出端口 2 → 方向 B→C
 SwC 去 SwA(lid1) → 出端口 2 → 方向 C→D
 SwD 去 SwB(lid3) → 出端口 2 → 方向 D→A
```

把这四股流量的占用方向画出来：

```
SwA 发往 SwC：A → B → C      占用 A→B、B→C
SwB 发往 SwD：B → C → D      占用 B→C、C→D
SwC 发往 SwA：C → D → A      占用 C→D、D→A
SwD 发往 SwB：D → A → B      占用 D→A、A→B
```

四股流量恰好都沿同一个旋转方向（顺时针）绕环。现在看拥塞时缓冲依赖如何成环：

```
  ┌──────────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                      │
  ↓                                                                                      │
 A→B 链路被占   等   → B→C 链路被占   等   → C→D 链路被占   等   → D→A 链路被占   等   → (回到 A→B)
```

逐环解释：

- A→B 上的流量（SwA→SwC）要前进，得等 **B→C** 腾空间；
- B→C 上的流量（SwB→SwD）要前进，得等 **C→D** 腾空间；
- C→D 上的流量（SwC→SwA）要前进，得等 **D→A** 腾空间；
- D→A 上的流量（SwD→SwB）要前进，得等 **A→B** 腾空间；
- 而 A→B 正被第一股占着......

四个"占用→等待"首尾相接，绕环整整一圈，构成完整的循环等待。谁都在等下一个先释放，谁都动不了——死锁。

### 为什么 min-hop 会埋这个雷

因为 min-hop 只认跳数，对每个目的地**独立**选最短路，完全不考虑这些选择叠加起来的全局方向。在这个环上，它给四股对角流量恰巧都选了顺时针方向，于是它们在环上首尾相接、凑成了一整圈同方向的依赖。min-hop 没有任何"全局方向约束"的概念，看不到这个隐患。

### 一个诚实的提醒

必须说清楚：上面这个死锁是**潜在风险**，不是说这四股流量一出现就立刻死锁。真正触发需要临界条件：网络拥塞到这些链路的缓冲区都被填满、且这几股流量恰好同时争抢。平时流量不饱和时，包正常流过，看不出任何问题。

但只要拓扑和路由**允许**这个循环依赖存在，它就是一颗定时炸弹：在足够高的负载、特定的流量组合下必然引爆。IB 的设计哲学是无损、追求确定性，绝不容忍这种"可能死锁"的隐患潜伏在网络里。这就是为什么需要下一节的 up-down。

## 12.5 Up-Down：给网络强加一个方向

up-down（updn）路由引擎解决 credit loop 的思路，是给整个网络**强加一个层级方向**。

### 核心思想

updn 的做法是：先指定一个（或一组）**根节点**，以根为基准，把每条链路定义出"上行"（朝根的方向）和"下行"（离根的方向）。然后施加一条铁律：**一个包一旦开始下行，就不允许再上行**。

这条"禁止下行转上行"的规则为什么能消除死锁？因为 credit loop 需要流量在环上凑成一整圈同方向的依赖，而这个循环必然包含"下行之后又上行"的转弯。把这种转弯一律禁掉，环就再也合不拢，因为循环依赖在物理上无法形成，死锁被从根源上铲除。代价是：某些流量不能再走它本来的最短路，必须绕行以遵守"先上后下"的规则。

### 为什么 updn 离不开根节点

这里有个关键点，正好能用一个真实的报错来说明。如果直接用 `-R updn` 而不指定根，OpenSM 会报错并放弃：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -R updn -f -'
...
Jun 18 04:37:25 404440 [49C006C0] 0x80 -> Entering MASTER state
Jun 18 04:37:25 411126 [49C006C0] 0x02 -> updn_lid_matrices: disabling UPDN algorithm, no root nodes were found
Jun 18 04:37:25 411293 [49C006C0] 0x01 -> ucast_mgr_route: updn: cannot build lid matrices.
Jun 18 04:37:25 411819 [49C006C0] 0x02 -> osm_ucast_mgr_process: minhop tables configured on all switches
Jun 18 04:37:25 435693 [49C006C0] 0x02 -> SUBNET UP
```

注意倒数第二行：updn 失败后**回退（fallback）成了 minhop**。这解释了一个容易踩的坑：如果不给根节点，你以为在跑 updn，实际跑的还是 minhop，转发表和 minhop 一模一样。

这个报错恰恰点破了 updn 的本质：**没有根，就没有"上"和"下"的概念，方向无从定义，算法根本无法工作。**

### 指定根节点，让 updn 真正运行

我们手动指定一个根。选 SwA（GUID 0x200000）当根，写一个 root guid 文件：

```bash
echo "0x200000" > ./updn-root.guids
```

然后用 `-R updn -a` 指定这个文件启动：

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -R updn -a ./updn-root.guids -f -'
```

这次日志里 updn 真正生效了，不再 fallback：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -R updn -a ./updn-root.guids -f -'
...

******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 18 04:49:36 664315 [FB4006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 18 04:49:36 684296 [FB4006C0] 0x80 -> Entering MASTER state
Jun 18 04:49:36 689869 [FB4006C0] 0x02 -> osm_topo_routing: Invalidating ucast cache due to crc files changes
Jun 18 04:49:36 692391 [FB4006C0] 0x02 -> osm_ucast_mgr_process: updn tables configured on all switches
Jun 18 04:49:36 721271 [FB4006C0] 0x02 -> SUBNET UP
```

`dump` 看 updn 算出的转发表：

```bash
expert@net21:~$ rlwrap ibsim -s ./ib2.net
parsing: ./ib2.net
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [566770] parse_port_connection_data: cannot parse remote lid and connection type
./ib2.net: parsed 40 lines
########################
Network simulator ready.
MaxNetNodes    = 2048
MaxNetSwitches = 256
MaxNetPorts    = 13312
MaxLinearCap   = 49152
MaxMcastCap    = 1024

sim> dump
# Net status - Thu Jun 18 04:49:44 2026

Switch 8 "SwA"  nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]3 [7]3 [8]3 [9]3
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "HcaA"[1]         4x  2.5G Active/LinkUp
200000  [2]     "SwB"[3]          4x  2.5G Active/LinkUp
200000  [3]     "SwD"[2]          4x  2.5G Active/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "SwB"  nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]3 [8]2 [9]3
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "HcaB"[1]         4x  2.5G Active/LinkUp
200001  [2]     "SwC"[3]          4x  2.5G Active/LinkUp
200001  [3]     "SwA"[2]          4x  2.5G Active/LinkUp
200001  [4]                       4x  2.5G Down/Polling
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Switch 8 "SwC"  nodeguid 200002 sysimgguid 200002
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
200002  [0]     "Sma Port"[0]    lid 6 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200002  [1]     "HcaC"[1]         4x  2.5G Active/LinkUp
200002  [2]     "SwD"[3]          4x  2.5G Active/LinkUp
200002  [3]     "SwB"[2]          4x  2.5G Active/LinkUp
200002  [4]                       4x  2.5G Down/Polling
200002  [5]                       4x  2.5G Down/Polling
200002  [6]                       4x  2.5G Down/Polling
200002  [7]                       4x  2.5G Down/Polling
200002  [8]                       4x  2.5G Down/Polling

Switch 8 "SwD"  nodeguid 200003 sysimgguid 200003
#       linearcap 49152 FDBtop 9 portchange 0
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]2 [6]3 [7]0 [8]3 [9]1
200003  [0]     "Sma Port"[0]    lid 7 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200003  [1]     "HcaD"[1]         4x  2.5G Active/LinkUp
200003  [2]     "SwA"[3]          4x  2.5G Active/LinkUp
200003  [3]     "SwC"[2]          4x  2.5G Active/LinkUp
200003  [4]                       4x  2.5G Down/Polling
200003  [5]                       4x  2.5G Down/Polling
200003  [6]                       4x  2.5G Down/Polling
200003  [7]                       4x  2.5G Down/Polling
200003  [8]                       4x  2.5G Down/Polling

Ca 2 "HcaA"     nodeguid 100000 sysimgguid 100000
100001  [1]     "SwA"[1]         lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaB"     nodeguid 100003 sysimgguid 100003
100004  [1]     "SwB"[1]         lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaC"     nodeguid 100006 sysimgguid 100006
100007  [1]     "SwC"[1]         lid 8 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100008  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "HcaD"     nodeguid 100009 sysimgguid 100009
10000a  [1]     "SwD"[1]         lid 9 lmc 0 smlid 1  4x  2.5G Active/LinkUp
10000b  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 8 nodes

```

## 12.6 对比：updn 改了哪几项，为什么非改不可

把 min-hop 和 updn 的转发表逐台对比，找出 updn 改动的表项。

```bash
# --- minhop ---
"SwA"  nodeguid 200000
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]2 [7]3 [8]3 [9]3
[0]     "Sma Port"[0]    lid 1
[1]     "HcaA"[1]
[2]     "SwB"[3]
[3]     "SwD"[2]

"SwB"  nodeguid 200001
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]2 [8]2 [9]3
[0]     "Sma Port"[0]    lid 3
[1]     "HcaB"[1]
[2]     "SwC"[3]
[3]     "SwA"[2]

"SwC"  nodeguid 200002
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
[0]     "Sma Port"[0]    lid 6
[1]     "HcaC"[1]
[2]     "SwD"[3]
[3]     "SwB"[2]

"SwD"  nodeguid 200003
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]3 [6]3 [7]0 [8]3 [9]1
[0]     "Sma Port"[0]    lid 7
[1]     "HcaD"[1]
[2]     "SwA"[3]
[3]     "SwC"[2]




# --- updn ---
"SwA"  nodeguid 200000
#       Forwarding table 0-9: [0]FF [1]0 [2]1 [3]2 [4]FF [5]2 [6]3 [7]3 [8]3 [9]3
[0]     "Sma Port"[0]    lid 1
[1]     "HcaA"[1]
[2]     "SwB"[3]
[3]     "SwD"[2]


"SwB"  nodeguid 200001
#       Forwarding table 0-9: [0]FF [1]3 [2]3 [3]0 [4]FF [5]1 [6]2 [7]3 [8]2 [9]3
[0]     "Sma Port"[0]    lid 3
[1]     "HcaB"[1]
[2]     "SwC"[3]
[3]     "SwA"[2]

"SwC"  nodeguid 200002
#       Forwarding table 0-9: [0]FF [1]2 [2]3 [3]3 [4]FF [5]3 [6]0 [7]2 [8]1 [9]2
[0]     "Sma Port"[0]    lid 6
[1]     "HcaC"[1]
[2]     "SwD"[3]
[3]     "SwB"[2]

"SwD"  nodeguid 200003
#       Forwarding table 0-9: [0]FF [1]2 [2]2 [3]2 [4]FF [5]2 [6]3 [7]0 [8]3 [9]1
[0]     "Sma Port"[0]    lid 7
[1]     "HcaD"[1]
[2]     "SwA"[3]
[3]     "SwC"[2]

```

以 SwA 为根，环上形成了清晰的层级：**SwA 在顶层（根），SwB 和 SwD 是中间层（各距根一跳），SwC 在底层（距根两跳，是离根最远的对角点）。**

底边那条经过 SwC 的路径，在 updn 看来就是"危险地带"，因为横穿底边往往意味着"下行后再上行"的转弯。

我们再逐项进行对比：

```bash
#  (2, 1) A ──────── B (3, 5)
#         │          │
#         │          │
#  (9, 7) D ──────── C (6, 8)

# --- minhop ---
 SwA 去 SwC(lid6) → 出端口 2 → 方向 A→B
 SwB 去 SwD(lid7) → 出端口 2 → 方向 B→C
 SwC 去 SwA(lid1) → 出端口 2 → 方向 C→D
 SwD 去 SwB(lid3) → 出端口 2 → 方向 D→A

# 在minhop中，观察 B/D 对角线的路由走向：
sim> route 3 7
From node "SwB" port 0 lid 3
[2] -> "SwC"[3]
[2] -> "SwD"[3]
To node "SwD" port 3 lid 7

sim> route 3 9
From node "SwB" port 0 lid 3
[3] -> "SwA"[2]
[3] -> "SwD"[2]
[1] -> "HcaD"[1]
To node "HcaD" port 1 lid 9

sim> route 5 7
From node "HcaB" port 1 lid 5
[1] -> "SwB"[1]
[2] -> "SwC"[3]
[2] -> "SwD"[3]
To node "SwD" port 3 lid 7

sim> route 5 9
From node "HcaB" port 1 lid 5
[1] -> "SwB"[1]
[3] -> "SwA"[2]
[3] -> "SwD"[2]
[1] -> "HcaD"[1]
To node "HcaD" port 1 lid 9

sim> route 7 3
From node "SwD" port 0 lid 7
[2] -> "SwA"[3]
[2] -> "SwB"[3]
To node "SwB" port 3 lid 3

sim> route 9 3
From node "HcaD" port 1 lid 9
[1] -> "SwD"[1]
[2] -> "SwA"[3]
[2] -> "SwB"[3]
To node "SwB" port 3 lid 3

sim> route 7 5
From node "SwD" port 0 lid 7
[3] -> "SwC"[2]
[3] -> "SwB"[2]
[1] -> "HcaB"[1]
To node "HcaB" port 1 lid 5

sim> route 9 5
From node "HcaD" port 1 lid 9
[1] -> "SwD"[1]
[3] -> "SwC"[2]
[3] -> "SwB"[2]
[1] -> "HcaB"[1]
To node "HcaB" port 1 lid 5



#  (2, 1) A ──────── B (3, 5)
#         │          │
#         │          │
#  (9, 7) D ──────── C (6, 8)

# --- updn ---
 SwA 去 SwC(lid6) → 出端口 3 → 方向 A→D
 SwB 去 SwD(lid7) → 出端口 3 → 方向 B→A
 SwC 去 SwA(lid1) → 出端口 2 → 方向 C→D
 SwD 去 SwB(lid3) → 出端口 2 → 方向 D→A

# 在updn中，观察 B/D 对角线的路由走向：：
sim> route 3 7
From node "SwB" port 0 lid 3
[3] -> "SwA"[2]
[3] -> "SwD"[2]
To node "SwD" port 2 lid 7

sim> route 3 9
From node "SwB" port 0 lid 3
[3] -> "SwA"[2]
[3] -> "SwD"[2]
[1] -> "HcaD"[1]
To node "HcaD" port 1 lid 9

sim> route 5 7
From node "HcaB" port 1 lid 5
[1] -> "SwB"[1]
[3] -> "SwA"[2]
[3] -> "SwD"[2]
To node "SwD" port 2 lid 7

sim> route 5 9
From node "HcaB" port 1 lid 5
[1] -> "SwB"[1]
[3] -> "SwA"[2]
[3] -> "SwD"[2]
[1] -> "HcaD"[1]
To node "HcaD" port 1 lid 9

sim> route 7 3
From node "SwD" port 0 lid 7
[2] -> "SwA"[3]
[2] -> "SwB"[3]
To node "SwB" port 3 lid 3

sim> route 9 3
From node "HcaD" port 1 lid 9
[1] -> "SwD"[1]
[2] -> "SwA"[3]
[2] -> "SwB"[3]
To node "SwB" port 3 lid 3

sim> route 7 5
From node "SwD" port 0 lid 7
[2] -> "SwA"[3]
[2] -> "SwB"[3]
[1] -> "HcaB"[1]
To node "HcaB" port 1 lid 5

sim> route 9 5
From node "HcaD" port 1 lid 9
[1] -> "SwD"[1]
[2] -> "SwA"[3]
[2] -> "SwB"[3]
[1] -> "HcaB"[1]
To node "HcaB" port 1 lid 5
```

这几处改动指向**同一个模式**：凡是 min-hop 选择"横穿底边、经过对角点 SwC"的路径，updn 都把它扳回"经过根 SwA"，例如：

- SwB(3) 去 SwD(7)：min-hop 横穿（经 SwC），updn 回根（经 SwA 再下到 SwD）
- SwD(7) 去 HcaB(5)：min-hop 横穿（经 SwC），updn 回根（经 SwA 再下到 SwB）

把这些改动对应回 12.4 那个 credit loop 就明白了：updn 改掉的，**正是构成那个循环依赖的危险转弯**。一旦所有跨环流量都遵守"先上行到根、再下行到目的"，环上就再也凑不齐一整圈同方向的依赖，死锁被消除。

而 SwC（对角点、底层）的转发表为什么不变？因为 SwC 在最底层，它去任何节点本来就是纯粹的"上行"，不存在"下行后再上行"的风险，updn 无需改动它。

### 进阶

这里我还想再强调一点，在这个只有四个节点的小环上，updn 的改动**并没有让任何路径的跳数变长**（都还是两跳）。它改变的是**路径方向的选择**，而非牺牲最短性。所以这个实验演示的是"updn 如何通过约束方向来消除环路依赖"，而不是"updn 为避环不惜绕远路"。

试想一下，如果是 6 台交换机组成的环，updn 为了遵守"先上后下"的规则，是否会让某些流量走比最短路更长的路径呢？

这个问题就留给读者自己写拓扑、自己测试啦。

## 12.7 小结

min-hop 和 up-down 代表了路由引擎设计上的两种取向：

**min-hop** 追求路径最优，每个目的地都走最短路，路径效率最高。但它没有全局方向约束，无法保证不形成 credit loop，在有环的拓扑上埋着潜在的死锁隐患。

**up-down** 追求安全，通过指定根、强加"先上后下"的方向规则，从根源上杜绝循环依赖，保证无死锁。代价是牺牲了一部分路径自由：某些流量不能走最短路，在大拓扑里甚至要明显绕行。
