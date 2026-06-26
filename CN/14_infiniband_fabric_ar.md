## 第十四章：Infiniband Fabric 自适应路由

### 14.1 静态路由的边界

在上一章中，我们通过实验观察了 updn 和 ftree 两种路由算法在同一 spine-leaf 拓扑下的行为差异。尽管两者在 LFT 生成逻辑上有所不同，但有一个根本性的共同点：**路由表在子网初始化时由 SM 一次性计算写入，此后固定不变。**

这意味着，SM 在计算路由时面对的是一张静止的拓扑图，而不是一个动态变化的流量矩阵。它能做的，只是在所有可能的路径中做一次静态的预分配。

这种方式在流量均匀分布时表现良好。但在实际的 AI 训练场景中，流量很少是均匀的。以 AllReduce 通信为例，当多个节点同时向同一目标发送梯度数据时，部分 Spine 会瞬间成为热点，而与此同时其他 Spine 可能几乎空闲。静态路由对此无能为力，即使某条链路已经拥塞，交换机仍然会按照 LFT 里写死的 port 继续转发，不会主动绕开。

所以，问题的本质在于：**静态路由的均衡粒度是"子网初始化时"，而流量的不均衡发生在"每个包转发时"**。这两个时间尺度之间的巨大落差，正是静态路由的根本局限。

---

### 14.2 自适应路由的思路

解决这个问题的思路并不复杂：既然 SM 在初始化时无法预知运行时的流量分布，那就把部分决策权下放给交换机，让它在转发每个包时，根据当前各 port 的实际负载状态来选择出口。

这就是自适应路由（Adaptive Routing，AR）的核心思想。

AR 并不是让 SM 退出路由决策，而是改变了 SM 与交换机之间的分工方式：

```
没有 AR：
  SM  →  计算每个 LID 走哪个 port  →  写入 LFT  →  交换机照表转发

有 AR：
  SM  →  计算哪些 port 是等价路径  →  写入 AR Group Table
  交换机  →  运行时感知各 port 负载  →  从 Group 内动态选 port
```

SM 不再告诉交换机"去 LID X 走 port 3"，而是告诉它"去 LID X 可以走 port 1、2、3、4，它们都是等价的，你自己看着办"。

交换机则在每次转发时，读取各候选 port 的实时 credit 或队列深度，选择当前最空闲的那个。

因此，决策的时间粒度，从"子网初始化时"缩短到了"每个包转发时"。

---

### 14.3 AR Group Table 的结构

AR 的实现载体是 AR Group Table，它与标准的 LFT 并列存在于交换机中。

标准 LFT 的结构是：

```
LID → 单一出口 port
```

AR Group Table 的结构是：

```
LID → AR Group ID → { port_1, port_2, port_3, ... }
```

SM 通过 ARM（Adaptive Routing Manager）模块在子网初始化时扫描所有支持 AR 的交换机，识别出等价的上行链路，将它们归入同一个 AR group，然后把 group 配置写入交换机。之后 SM 不再参与逐包的转发决策，这部分完全由交换机硬件自主完成。

在上一章的 spine-leaf 拓扑中，每个 Leaf switch 有 4 条上行链路分别连到 4 个 Spine，AR group 的配置大致如下：

```
Leaf1 AR Group 0：{ port1, port2, port3, port4 }   ← 4条上行等价链路
LFT[lid_X] → Group 0    （所有跨 Leaf 的目标 LID 都指向这个 group）
```

交换机转发时，不再查 LFT 得到一个固定 port，而是查到 Group 0，再实时从 4 个 port 中选一个负载最低的转发出去。

---

### 14.4 AR 的代价：包乱序问题

AR 并不是没有代价的。IB 协议在设计上保证了同一 QP 内的消息严格有序，发送方按顺序发出的包，接收方按同样的顺序收到。这个保证的前提是：同一条连接的所有包走同一条路径。

AR 打破了这个前提。同一个 QP 的连续两个包，可能因为 port 负载的瞬时变化而走了不同的 Spine，经历不同的排队延迟，以乱序到达接收方。

为了支持 AR，HCA 需要具备包重排序能力（Packet Reordering），在接收端将乱序到达的包重新排列后再交给上层。这对 HCA 的硬件设计提出了额外要求，也是早期 IB 设备不支持 AR 的原因之一。现代 NVIDIA ConnectX 系列 HCA 均已支持包重排序，AR 才得以在生产环境中广泛部署。

> AR 的转发决策发生在交换机处理每个数据包时，理论粒度为逐包（per-packet）。但由于 IB RC QP 的保序要求，实际部署中 HCA 需要具备包重排序能力，NVIDIA ConnectX 系列 HCA 均已支持。NVIDIA 的具体实现细节属于私有技术，是否在某些场景下以 flow(QP) 为粒度做保守决策，目前没有公开的完整说明。

---

### 14.5 实验测试

接下来，我们通过实验观察 ar_ftree 与 ftree 在控制平面上的差异。不过有些遗憾的是，ibsim 的虚拟交换机并不支持 Adaptive Routing，AR 依赖 NVIDIA IB 交换机的私有扩展能力，ibsim 没有实现这部分。因此我们能观察到的，只限于 OpenSM 控制平面的行为，而看不到交换机数据平面的动态决策过程。尽管如此，这个实验本身也能说明一些重要的事情。

首先，如往常一样启动 ibsim：

```bash
expert@net21:~$ rlwrap ibsim -s ./ib3.net
```

然后，启动 openSM，指定 ar_ftree 路由引擎：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -R ar_ftree -f -'
[sudo] password for expert:
-------------------------------------------------
OpenSM 5.21.12.MLNX20250617.f74e01b8
Command Line Arguments:
 Activate 'ar_ftree' routing engine(s)
 Log File: -
-------------------------------------------------
Jun 21 13:03:06 711749 [CE4D4740] 0x03 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
Jun 21 13:03:06 712028 [CE4D4740] 0x80 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
ibwarn: [11064] sim_connect: attached as client 0 at node "Spine1"
Jun 21 13:03:06 730762 [CE4D4740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 21 13:03:06 731135 [CE4D4740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 21 13:03:06 731397 [CE4D4740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Using default GUID 0x200000
Jun 21 13:03:06 745060 [CE4D4740] 0x02 -> osm_tenant_mgr_init: tenant mgr is disabled
Jun 21 13:03:06 745794 [CE4D4740] 0x02 -> osm_issu_mgr_init: issu_mgr is initialized
Jun 21 13:03:06 745979 [CE4D4740] 0x80 -> Entering DISCOVERING state
Jun 21 13:03:06 747846 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x81 binding to port GUID 0x200000
Jun 21 13:03:06 759908 [CE4D4740] 0x02 -> osm_sm_bind: Bind to port guid 0x200000, port index 0 as main SM port
Jun 21 13:03:06 760050 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x03 binding to port GUID 0x200000
Jun 21 13:03:06 770072 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x04 binding to port GUID 0x200000
Jun 21 13:03:06 770205 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x21 binding to port GUID 0x200000
Jun 21 13:03:06 770288 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0a binding to port GUID 0x200000
Jun 21 13:03:06 770369 [CE4D4740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0c binding to port GUID 0x200000
Jun 21 13:03:06 770482 [CE4D4740] 0x02 -> osm_opensm_bind: Setting IS_SM on port 0x0000000000200000
OpenSM $ Jun 21 13:03:06 771114 [BE3ED6C0] 0x02 -> do_sweep:


******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 21 13:03:06 771282 [BE3ED6C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 21 13:03:06 803001 [BE3ED6C0] 0x80 -> Entering MASTER state
Jun 21 13:03:06 813008 [BE3ED6C0] 0x02 -> fabric_dump_general_info: General fabric topology info
Jun 21 13:03:06 813137 [BE3ED6C0] 0x02 -> fabric_dump_general_info: ============================
Jun 21 13:03:06 813220 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - FatTree rank (roots to leaf switches): 2
Jun 21 13:03:06 813343 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - FatTree max switch rank: 1
Jun 21 13:03:06 813379 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - Fabric has 0 Routers which are considered as IO nodes
Jun 21 13:03:06 813416 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - Fabric has 16 CAs, 16 CA ports (16 of them CNs), 8 switches
Jun 21 13:03:06 813448 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - Fabric has 4 switches at rank 0 (roots)
Jun 21 13:03:06 813474 [BE3ED6C0] 0x02 -> fabric_dump_general_info:   - Fabric has 4 switches at rank 1 (4 of them leafs)
AR Manager - Configuration cycle (number 1) completed successfully
Jun 21 13:03:06 837934 [BE3ED6C0] 0x02 -> osm_ucast_mgr_process: ar_ftree tables configured on all switches
Jun 21 13:03:06 838829 [BE3ED6C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 21 13:03:06 910438 [BE3ED6C0] 0x02 -> SUBNET UP

OpenSM $
OpenSM $
```

注意，关键日志输出如下：

```bash
fabric_dump_general_info:   - FatTree rank (roots to leaf switches): 2
fabric_dump_general_info:   - Fabric has 4 switches at rank 0 (roots)
fabric_dump_general_info:   - Fabric has 4 switches at rank 1 (4 of them leafs)
AR Manager - Configuration cycle (number 1) completed successfully
osm_ucast_mgr_process: ar_ftree tables configured on all switches
osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
SUBNET UP
```

与上一章跑普通 ftree 时相比，日志多出了两行：

```
AR Manager - Configuration cycle (number 1) completed successfully
osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
```

**第一行**说明 AR Manager（ARM）模块被激活并完成了一次配置周期。ARM 是 OpenSM 中负责 AR 配置的组件，它在路由计算完成后扫描 fabric 中所有的交换机，识别支持 AR 的设备，并向其写入 AR Group Table。"completed successfully"说明这个流程正常走完了。但是请注意："成功完成"只意味着流程没有报错，并不意味着 AR 配置真的生效了。

**第二行**才是真正的结论：`No fabric switch supports pFRN`。pFRN（port-based Forwarding Route Notification）是 AR 的配套功能，ARM 通过查询这个能力位来判断交换机是否支持 AR。这里明确说明，fabric 中没有任何一台交换机声明了该能力。换句话说，ARM 扫描了所有虚拟交换机，发现它们全都不具备 AR 硬件能力，于是跳过了 AR Group Table 的写入，什么也没配置。

此时如果用 `ibroute` 查看各 Leaf switch 的转发表，会发现结果与普通 ftree 完全相同——这正是预期的结果，因为在没有 AR Group Table 的情况下，数据包仍然按照普通 LFT 转发，ar_ftree 退化成了 ftree。

---

### 14.6 ibsim 的边界

这个实验清楚地划出了 ibsim 能做什么、不能做什么。

**能观察到的（控制平面）：**

OpenSM 识别 fat-tree 拓扑结构、计算路由、生成 LFT、调用 ARM 模块、尝试配置 AR Group Table——这整个控制平面流程，在 ibsim 中都能完整运行和观察。日志里的拓扑识别信息（rank、leaf/root 数量）、ARM 的启动和运行记录，都是真实的控制平面行为。

**观察不到的（数据平面）：**

| 想观察的                              | ibsim 能否做到                      |
| ------------------------------------- | ----------------------------------- |
| AR Group Table 的实际内容             | ❌ 虚拟 switch 不支持 AR capability |
| Switch 运行时从 group 内选了哪个 port | ❌ ibsim 没有数据平面               |
| AR 实际减少了多少拥塞                 | ❌                                  |
| port 负载超阈值时的切换行为           | ❌                                  |
| 包乱序（out-of-order）的发生率        | ❌                                  |

要观察这些，需要真实的 NVIDIA IB 交换机，配合 UFM 的 AR 监控界面或 `ibdiagnet --ar_groups` 命令，才能看到 AR Group Table 的配置内容和运行时的 port 选择统计。

---

### 14.7 小结

通过上一章和本章的实验，我们完整地走过了 IB 路由从静态到动态的演进路径。

updn 和 ftree 都属于静态路由：SM 在子网初始化时计算路由、写入 LFT，之后路由表固定不变。两者的区别在于计算逻辑：updn 依赖通用的最短路径计数器，ftree 则显式感知 fat-tree 的层次结构，在对称拓扑下能产生更规则、更可预测的路由分布。

ar_updn 和 ar_ftree 在此基础上引入了自适应路由：SM 额外计算哪些 port 是等价的上行路径，通过 ARM 写入 AR Group Table；交换机在转发每个包时，从 group 内实时选择负载最低的 port。决策的时间粒度从"子网初始化时"缩短到了"每个包转发时"，这是应对 AI 训练场景下突发性流量不均衡的根本手段。

ibsim 让我们完整观察了控制平面的行为，也清楚地标出了它的边界。AR 真正的价值体现在数据平面：当 AllReduce、All-to-All 等集合通信在数千个 GPU 之间同时进行时，AR 对实时流量的感知和动态均衡，才是 IB 网络能够充分发挥能力的关键所在。
