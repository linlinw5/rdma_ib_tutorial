# 第九章: InfiniBand 实验环境搭建

## 9.1 概述

IB 普及率低，最大的一个问题是硬件不通用，我手里没有任何 IB 的交换机和网卡，但这并不妨碍我们继续探索 Infiniband。

本章介绍在无真实 IB 硬件的情况下，使用 **ibsim**（fabric 模拟器）+ **opensm**（Subnet Manager）搭建完整 IB 实验环境的过程。

**成功验证的版本组合：**

| 组件   | 版本                 |
| ------ | -------------------- |
| OS     | Ubuntu 24.04         |
| opensm | 5.21.12.MLNX20250617 |
| ibsim  | 0.12                 |
| 软件源 | NVIDIA DOCA 2.9.4    |

本章以环境搭建为主，简要罗列基础测试的命令，后续章节会详细讲述。

---

## 9.2 ibsim 和 openSM 介绍

**ibsim**

ibsim 是一个 InfiniBand 子网模拟器，其核心作用是在单台 Linux 主机上虚拟出一个完整的 IB 子网拓扑，而无需任何物理 HCA 硬件。它通过创建虚拟的 umad（用户态 MAD）设备来对外暴露接口，使得所有标准的 IB 管理工具（包括 OpenSM、ibnetdiscover、ibtracert 等）都能以为在操作真实硬件的方式与之交互。ibsim 从一个拓扑描述文件读入节点和链路定义，启动后即维护一张模拟的 MAD 响应表，能够响应 SMP（子网管理包）中的 NodeInfo、PortInfo、SwitchInfo、LinearForwardingTable 等属性查询，也能够接受 OpenSM 下发的 LID 分配和 LFT 写入操作。

对于开发、测试和教学场景而言，ibsim 的最大价值在于它将"拓扑规模"与"硬件成本"彻底解耦。我们可以在一台笔记本上模拟包含数十个交换机和数百个端口的 fat-tree 网络，观察 OpenSM 如何发现拓扑、计算路由、分配 LID，并通过 umad 抓包追踪每一条 Directed Route SMP 的往返路径，而全程不需要一块真实的 Mellanox HCA。

**OpenSM**

OpenSM 是 InfiniBand 规范定义的子网管理器（Subnet Manager）的开源实现，是 IB 子网能够正常工作的必要条件。在 IB 架构中，SM 是整个子网的"控制平面大脑"：它负责拓扑发现（通过 Directed Route SMP 遍历所有节点）、LID 分配、路由计算（LFT/MFT 下发）、以及子网从 Discovering 到 Armed 再到 Active 的状态机推进。

OpenSM 以用户态进程的形式运行，通过 /dev/infiniband/umad\* 字符设备收发 MAD 报文。

在部署方面，OpenSM 具有相当的灵活性：在我们的实验环境中，它直接运行在普通 Linux 主机上，与 ibsim 配合管理模拟子网；而在生产环境中，openSM 通常内嵌运行在 IB 交换机固件内部，Mellanox/NVIDIA 的 IB 交换机即以这种方式自带 SM 功能，无需外部主机介入即可完成子网的自治管理。

**模拟环境实现原理**

我们的实验环境就是一台 Ubuntu Linux，上面安装了 ibsim 和 openSM，他们之间的通信原理如下：

```
opensm / 诊断工具
      ↓
  libibumad          ← 用户态 MAD 访问库（LD = Linker/Loader）
      ↓
  LD_PRELOAD=libumad2sim.so   ← 拦截 umad 调用，重定向到 ibsim
      ↓
  ibsim              ← 模拟 IB fabric 的 MAD 响应
```

```bash
# `libumad2sim.so` 重新定义了同名函数
libibumad 提供：    umad_open()  umad_send()  umad_recv() ...
libumad2sim 也提供：umad_open()  umad_send()  umad_recv() ...（同名）
```

通过环境变量设置（`export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so`），动态链接器解析函数符号时先找到 `libumad2sim` 里的版本，`libibumad` 里的同名函数就被**遮蔽**了。

`libumad2sim` 内部实现里，把这些函数的行为从"读写 `/dev/infiniband/umadX`"改成了"读写 ibsim 的 unix socket"。

从而实验将 opensm 和诊断工具的 MAD 报文重定向到 ibsim 的 unix socket，无需真实硬件，无需重新编译任何程序。

---

## 9.3 安装

openSM 和 ibsim 本身都是开源软件，直接在干净的 ubuntu 系统上，用`sudo apt install`就可以了。

为了便捷，以及演示如何安装 MLNX_OFED，接下来我们使用 NVIDIA DOCA apt 源一键安装。

网址：`https://linux.mellanox.com/public/repo/doca`

doca-ofed 是元包，会自动拉取所有依赖（opensm、ibsim、infiniband-diags、libibmad5、libibumad3 等）

> 版本说明：DOCA 2.9.4 对应 opensm 5.21.12，与 ibsim 0.12 兼容。

> 测试中，我发现：DOCA 3.3.0 对应 opensm 5.26.x，与 ibsim 0.12 存在兼容性问题，SM 会卡死在 DISCOVERING 阶段。

```bash
# 添加 NVIDIA DOCA 2.9.4 源
curl https://linux.mellanox.com/public/repo/doca/GPG-KEY-Mellanox.pub \
  | gpg --dearmor \
  | sudo tee /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub > /dev/null

echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub] \
  https://linux.mellanox.com/public/repo/doca/2.9.4/ubuntu24.04/x86_64/ ./" \
  | sudo tee /etc/apt/sources.list.d/doca.list

sudo apt-get update

# 一键安装（自动处理所有依赖）
sudo apt-get install doca-ofed -y

# rlwrap：为 ibsim 控制台提供退格键和历史记录支持
sudo apt-get install rlwrap -y
```

安装完成后以下命令均可用：

| 命令                                            | 来源包           |
| ----------------------------------------------- | ---------------- |
| `opensm`                                        | opensm           |
| `ibsim`                                         | ibsim            |
| `ibnetdiscover`、`ibnodes`、`ibswitches`        | infiniband-diags |
| `smpquery`、`saquery`、`dump_lfts`、`ibtracert` | infiniband-diags |

---

## 9.4 拓扑文件

ibsim 使用文本文件描述 fabric 拓扑，格式与 `ibnetdiscover` 输出兼容。

### 格式规范

```
# 节点头部
type(Switch|Hca)  port_count  "nodeid"

# 端口连接
[localport]  "remoteid"[remoteport]
```

### 示例：双交换机拓扑（net-examples/net）

```
Switch 8 "Switch1"
[1]     "Hca1"[1]
[2]     "Hca2"[2]
[3]     "Switch2"[3]
[4]     "Switch2"[4]

Switch 8 "Switch2"
[3]     "Switch1"[3]
[4]     "Switch1"[4]

Hca 2 "Hca1"
[1]     "Switch1"[1]

Hca 2 "Hca2"
[2]     "Switch1"[2]
```

> 初次使用，建议直接使用 ibsim 自带的示例拓扑文件：
> `/usr/share/doc/ibsim/net-examples/net`

每个 Switch 的 port 0 是 **SMA Port**（Subnet Management Agent），这是交换机内置的管理逻辑端口，永远处于 Active 状态，负责响应 SM 发来的 SMP MAD，它不是物理端口，没有物理链路。

---

## 9.5 启动流程

启动 openSM 和 ibsim 需要用到三个独立的 terminal 窗口，分别用于：ibsim 控制台、openSM 控制台、运行管理工具。

### Terminal 1：启动 ibsim

```bash
# rlwrap 提供退格键、上下翻滚历史记录等 readline 功能（ibsim 原生不支持）
rlwrap ibsim -s -v /usr/share/doc/ibsim/net-examples/net
```

参数说明：

| 参数         | 说明                                    |
| ------------ | --------------------------------------- |
| `-s`         | 立即启动网络，不等待控制台 `Start` 命令 |
| `-v`         | verbose 模式，打印 MAD 收发详情         |
| 最后一个参数 | 拓扑文件路径                            |

启动成功后显示：

```bash
expert@net21:~$ rlwrap ibsim -s -v /usr/share/doc/ibsim/net-examples/net
parsing: /usr/share/doc/ibsim/net-examples/net
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [6813] parse_port_connection_data: cannot parse remote lid and connection type
/usr/share/doc/ibsim/net-examples/net: parsed 39 lines
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [6813] connect_ports: 8 ports connected
########################
Network simulator ready.
MaxNetNodes    = 2048
MaxNetSwitches = 256
MaxNetPorts    = 13312
MaxLinearCap   = 49152
MaxMcastCap    = 1024
sim>
```

### 验证拓扑

```bash
sim> dump
# Net status - Sat Jun 20 14:41:29 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 0 portchange 1
200000  [0]     "Sma Port"[0]    lid 0 lmc 0 smlid 0  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Init/LinkUp
200000  [2]     "Hca2"[2]         4x  2.5G Init/LinkUp
200000  [3]     "Switch2"[3]      4x  2.5G Init/LinkUp
200000  [4]     "Switch2"[4]      4x  2.5G Init/LinkUp
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 0 portchange 1
200001  [0]     "Sma Port"[0]    lid 0 lmc 0 smlid 0  4x  2.5G Active/LinkUp
200001  [1]                       4x  2.5G Down/Polling
200001  [2]                       4x  2.5G Down/Polling
200001  [3]     "Switch1"[3]      4x  2.5G Init/LinkUp
200001  [4]     "Switch1"[4]      4x  2.5G Init/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
100005  [2]     "Switch1"[2]     lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
#  dumped 4 nodes
```

确认所有节点和连接正确，端口显示 `Init/LinkUp`。

### Terminal 2：启动 opensm

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -f -'
```

参数说明：

| 参数                 | 说明                                                                        |
| -------------------- | --------------------------------------------------------------------------- |
| `sudo bash -c '...'` | 在 sudo 子 shell 内设置环境变量，确保 `LD_PRELOAD` 不被 sudo 清除           |
| `LD_PRELOAD=...`     | 预加载 `libumad2sim.so`，将 umad 调用重定向到 ibsim                         |
| `-q local`           | 为 openSM 启用本地控制台（可选）                                            |
| `-f -`               | 日志输出到 stdout（`-` 表示标准输出），也可以将日志输出到文件，详见帮助文档 |

备注：opemSM 的控制台在本实验中用处不大，所以 `-q` 参数也可以省略。

SM 正常收敛后，opensm 输出：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -f -'
-------------------------------------------------
OpenSM 5.21.12.MLNX20250617.f74e01b8
Command Line Arguments:
 d level = 0x2
 Debug mode: Force Log Flush
 Log File: -
-------------------------------------------------
Jun 20 14:38:48 734296 [F66B8740] 0x03 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
Jun 20 14:38:48 734543 [F66B8740] 0x80 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
ibwarn: [6823] sim_connect: attached as client 0 at node "Switch1"
Jun 20 14:38:48 754980 [F66B8740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 20 14:38:48 755302 [F66B8740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 20 14:38:48 755577 [F66B8740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Using default GUID 0x200000
Jun 20 14:38:48 769904 [F66B8740] 0x02 -> osm_tenant_mgr_init: tenant mgr is disabled
Jun 20 14:38:48 770683 [F66B8740] 0x02 -> osm_issu_mgr_init: issu_mgr is initialized
Jun 20 14:38:48 770912 [F66B8740] 0x80 -> Entering DISCOVERING state
Jun 20 14:38:48 772295 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x81 binding to port GUID 0x200000
Jun 20 14:38:48 785724 [F66B8740] 0x02 -> osm_sm_bind: Bind to port guid 0x200000, port index 0 as main SM port
Jun 20 14:38:48 785879 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x03 binding to port GUID 0x200000
Jun 20 14:38:48 796730 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x04 binding to port GUID 0x200000
Jun 20 14:38:48 796878 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x21 binding to port GUID 0x200000
Jun 20 14:38:48 796991 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0a binding to port GUID 0x200000
Jun 20 14:38:48 797097 [F66B8740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0c binding to port GUID 0x200000
Jun 20 14:38:48 797202 [F66B8740] 0x02 -> osm_opensm_bind: Setting IS_SM on port 0x0000000000200000
OpenSM $ Jun 20 14:38:48 798594 [E63ED6C0] 0x02 -> do_sweep:


******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 20 14:38:48 799183 [E63ED6C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 20 14:38:48 817351 [E63ED6C0] 0x80 -> Entering MASTER state
AR Manager - Configuration cycle (number 1) completed successfully
Jun 20 14:38:48 850162 [E63ED6C0] 0x02 -> osm_ucast_mgr_process: ar_updn tables configured on all switches
Jun 20 14:38:48 850786 [E63ED6C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 20 14:38:48 886400 [E63ED6C0] 0x02 -> SUBNET UP

OpenSM $
OpenSM $
```

### 验证 ibsim 变化

opensm 启动后，在 ibsim 控制台用 `Attached` 命令可以确认连接：

```bash
# 关闭日志打印
sim> verbose 0
simulator verbose level is 0

sim> attached
Client 0: pid 6877 connected at "Switch1" port 0x200000, lid 1, qp 0 SM
# opensm 已成功连接，挂在 Switch1 的管理端口（port 0，GUID 0x200000）上，并被分配了 LID 1
```

再次查看ibsim 的 `Dump` ，应显示：

- 所有有连线的端口从 `Init` 变为 `Active`
- 每个端口分配了 LID
- 交换机显示 `Forwarding table`，示例如下：

```bash
sim> dump
# Net status - Sat Jun 20 14:40:37 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 4 portchange 0
#       Forwarding table 0-4: [0]FF [1]0 [2]1 [3]3 [4]2
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Active/LinkUp
200000  [2]     "Hca2"[2]         4x  2.5G Active/LinkUp
200000  [3]     "Switch2"[3]      4x  2.5G Active/LinkUp
200000  [4]     "Switch2"[4]      4x  2.5G Active/LinkUp
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 4 portchange 0
#       Forwarding table 0-4: [0]FF [1]3 [2]4 [3]0 [4]3
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]                       4x  2.5G Down/Polling
200001  [2]                       4x  2.5G Down/Polling
200001  [3]     "Switch1"[3]      4x  2.5G Active/LinkUp
200001  [4]     "Switch1"[4]      4x  2.5G Active/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
100005  [2]     "Switch1"[2]     lid 4 lmc 0 smlid 1  4x  2.5G Active/LinkUp
#  dumped 4 nodes
```

### Terminal 3：运行管理工具

```bash
# 先设置环境变量
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so

# 完整拓扑发现
expert@net21:~$ ibnetdiscover
ibwarn: [6931] sim_connect: attached as client 1 at node "Switch1"
#
# Topology file: generated on Sat Jun 20 14:51:33 2026
#
# Initiated from node 0000000000200000 port 0000000000200000

vendid=0x0
devid=0x0
sysimgguid=0x200001
switchguid=0x200001(200001)
Switch  8 "S-0000000000200001"          # "Switch2" base port 0 lid 3 lmc 0
[3]     "S-0000000000200000"[3]         # "Switch1" lid 1 4xSDR
[4]     "S-0000000000200000"[4]         # "Switch1" lid 1 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x200000
switchguid=0x200000(200000)
Switch  8 "S-0000000000200000"          # "Switch1" base port 0 lid 1 lmc 0
[1]     "H-0000000000100000"[1](100001)                 # "Hca1" lid 2 4xSDR
[2]     "H-0000000000100003"[2](100005)                 # "Hca2" lid 4 4xSDR
[3]     "S-0000000000200001"[3]         # "Switch2" lid 3 4xSDR
[4]     "S-0000000000200001"[4]         # "Switch2" lid 3 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x100003
caguid=0x100003
Ca      2 "H-0000000000100003"          # "Hca2"
[2](100005)     "S-0000000000200000"[2]         # lid 4 lmc 0 "Switch1" lid 1 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x100000
caguid=0x100000
Ca      2 "H-0000000000100000"          # "Hca1"
[1](100001)     "S-0000000000200000"[1]         # lid 2 lmc 0 "Switch1" lid 1 4xSDR
```

---

## 9.6 基础测试

### IB 网络基础信息查询

```bash
# 查看所有 IB 交换机
expert@net21:~$ ibswitches
ibwarn: [6937] sim_connect: attached as client 1 at node "Switch1"
Switch  : 0x0000000000200001 ports 8 "Switch2" base port 0 lid 3 lmc 0
Switch  : 0x0000000000200000 ports 8 "Switch1" base port 0 lid 1 lmc 0

# 查所有节点 LID/GUID
expert@net21:~$ ibnodes
ibwarn: [6944] sim_connect: attached as client 1 at node "Switch1"
Ca      : 0x0000000000100003 ports 2 "Hca2"
Ca      : 0x0000000000100000 ports 2 "Hca1"
ibwarn: [6950] sim_connect: attached as client 1 at node "Switch1"
Switch  : 0x0000000000200001 ports 8 "Switch2" base port 0 lid 3 lmc 0
Switch  : 0x0000000000200000 ports 8 "Switch1" base port 0 lid 1 lmc 0
```

### SMP 直接查询

直接向节点发 SMP MAD，不经过 SA：

```bash
# 查端口信息（LID 2，port 1）
smpquery portinfo 2 1

# 查节点信息
smpquery nodeinfo 2

# 查交换机信息（LID 1，port 0）
smpquery switchinfo 1 0

# 打印所有交换机的线性转发表
dump_lfts
```

### SA 查询

SA（Subnet Administration）是上层应用（MPI、RDMA）查询路径的接口：

```bash
# 查所有节点记录
saquery -N

# 查所有端口记录
saquery -P

# 查路径（LID 2 到 LID 4 的 PathRecord）
# PathRecord 包含 SL、MTU、速率等，RDMA 建连前必须查询
saquery -p --slid 2 --dlid 4

# 查 SM 信息
saquery -s
```

### 路径追踪

```bash
# 追踪 LID 2 到 LID 4 的逐跳路径
ibtracert 2 4

# ibsim 内部路由验证
sim> Route 2 4
```

### 拓扑变化测试

在 ibsim 控制台操作：

```bash
# 断开一条 inter-switch link，观察 SM 重新扫描网络，并收敛
sim> Unlink "Switch1"[4]

# 恢复链路，再次观察收敛
sim> ReLink "Switch1"[4]

# 断开整个节点
sim> Unlink "Hca1"

# 恢复
sim> ReLink "Hca1"
```

---

## 9.7 ibsim 控制台命令速查

| 命令                                 | 说明                                |
| ------------------------------------ | ----------------------------------- |
| `Dump ["nodeid"]`                    | 打印节点/全网状态，包含 LID、LFT    |
| `Route <from-lid> <to-lid>`          | 验证两个 LID 之间的路由路径         |
| `Unlink "nodeid"[port]`              | 断开端口连接（模拟链路故障）        |
| `Unlink "nodeid"`                    | 断开节点所有链路                    |
| `ReLink "nodeid"[port]`              | 恢复端口连接                        |
| `ReLink "nodeid"`                    | 恢复节点所有链路                    |
| `Clear "nodeid"[port]`               | 断开并重置端口状态                  |
| `Error "nodeid"[port] <rate> [attr]` | 注入丢包错误                        |
| `Attached`                           | 列出当前连接的客户端                |
| `Verbose [level]`                    | 设置 verbose 级别（0=静默，1=调试） |
| `Start`                              | 启动网络（不加 -s 参数时手动启动）  |
| `Quit`                               | 退出 ibsim                          |

---

## 9.8 常见问题

### SM 卡死在 DISCOVERING 状态

**原因**：opensm 版本过新（5.26.x），与 ibsim 0.12 不兼容，SM 在 SMInfo 选举阶段卡死。

**解决**：降级到 opensm 5.21.12。

### ibsim/openSM 控制台不支持退格键

**原因**：ibsim 控制台未使用 readline 库，退格键输出 `^H` 而非真正删除字符。

**解决**：使用 `rlwrap ibsim ...` 启动。

### ibsim/openSM 如何退出

ibsim 直接在控制台用 `quit` 退出即可；

openSM 需要在新的 terminal 中用 `sudo pkill -9 opensm` 强制退出。

生产环境中 OpenSM 应该作为 systemd service 运行，结合 `systemctl` 来管理，或者直接跑在 IB 交换机上，具体可以参考相关支持文档。
