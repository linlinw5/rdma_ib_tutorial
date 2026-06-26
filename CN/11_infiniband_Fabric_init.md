# 第十一章：InfiniBand Fabric 初始化

## 11.1 从"插好线"到"能通信"之间

对于 IB 来说，把所有线缆插好，设备之间就有了物理连通，但这还不是一张能用的网络。此时 IB Fabric 的每个端口还没有地址，每台交换机还不知道该把包往哪转，整个 fabric 是一堆互相连着、却彼此不认识的设备。

从这个状态，到一张每个端口都有 LID、每台交换机都有转发表、任意两点都能通信的网络，中间做事的就是子网管理器（SM）。在真实硬件上，这个过程在你给设备上电后的一两秒内自动完成，快得你根本看不见。这一章我们借助前面搭好的 ibsim + opensm 环境，把这个过程慢放下来，一步步看清楚 SM 到底做了什么。

这一章的目标，就是揭开 OpenSM 的神秘面纱，你会看到，SM 本质就是一个"发现拓扑 → 编号 → 算路由 → 下发 → 持续监控"的循环。

## 11.2 演示拓扑与运行环境

本章所有演示都基于这样一张拓扑：两台交换机互联，各自挂一个 HCA。

ibsim拓扑文件如下，存放于 `~/ib1.net`：

```bash
# Hca1 [1]─────────[1] Switch1 [3]────────[4] Switch2 [1]─────────[1] Hca2

Switch  8 "Switch1"
[1]     "Hca1"[1]
[3]     "Switch2"[4]


Switch 8 "Switch2"
[1]     "Hca2"[1]
[4]     "Switch1"[3]


Hca     2 "Hca1"
[1]     "Switch1"[1]

Hca     2 "Hca2"
[1]     "Switch2"[1]
```

在terminal中启动ibsim，开启模拟fabric，可以看到如下日志：

```bash
expert@net21:~$ rlwrap ibsim -s -v ./ib1.net
parsing: ./ib1.net
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
ibwarn: [564977] parse_port_connection_data: cannot parse remote lid and connection type
./ib1.net: parsed 15 lines
ibwarn: [564977] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [564977] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [564977] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [564977] encode_trap128: switch trap 128 for lid 0 with smlid 0
ibwarn: [564977] connect_ports: 6 ports connected
########################
Network simulator ready.
MaxNetNodes    = 2048
MaxNetSwitches = 256
MaxNetPorts    = 13312
MaxLinearCap   = 49152
MaxMcastCap    = 1024
sim>
sim> dump
# Net status - Wed Jun 17 16:45:30 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 0 portchange 1
200000  [0]     "Sma Port"[0]    lid 0 lmc 0 smlid 0  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Init/LinkUp
200000  [2]                       4x  2.5G Down/Polling
200000  [3]     "Switch2"[4]      4x  2.5G Init/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 0 portchange 1
200001  [0]     "Sma Port"[0]    lid 0 lmc 0 smlid 0  4x  2.5G Active/LinkUp
200001  [1]     "Hca2"[1]         4x  2.5G Init/LinkUp
200001  [2]                       4x  2.5G Down/Polling
200001  [3]                       4x  2.5G Down/Polling
200001  [4]     "Switch1"[3]      4x  2.5G Init/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Switch2"[1]     lid 0 lmc 0 smlid 0  4x  2.5G Init/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 4 nodes
```

从以上日志中，通过 dump ，我们可以看到所有的设备lid均为 0，因为这个时候 SM 还没有上线。

接下来，开启一个新的terminal，用于OpenSM。OpenSM 通过 ibsim 提供的 `umad2sim` 预加载库，挂接到模拟 fabric 上运行，启动后日志开头能看到：

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -f -'
[sudo] password for expert:
-------------------------------------------------
OpenSM 5.21.12.MLNX20250617.f74e01b8
Command Line Arguments:
 d level = 0x2
 Debug mode: Force Log Flush
 Log File: -
-------------------------------------------------
Jun 17 16:47:41 376915 [50493740] 0x03 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
Jun 17 16:47:41 377247 [50493740] 0x80 -> OpenSM 5.21.12.MLNX20250617.f74e01b8
ibwarn: [564980] sim_connect: attached as client 0 at node "Switch1"
Jun 17 16:47:41 392446 [50493740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 17 16:47:41 392763 [50493740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Jun 17 16:47:41 393119 [50493740] 0x02 -> osm_vendor_init: 1000 pending umads specified
Using default GUID 0x200000
Jun 17 16:47:41 405789 [50493740] 0x02 -> osm_tenant_mgr_init: tenant mgr is disabled
Jun 17 16:47:41 406589 [50493740] 0x02 -> osm_issu_mgr_init: issu_mgr is initialized
Jun 17 16:47:41 406790 [50493740] 0x80 -> Entering DISCOVERING state
Jun 17 16:47:41 408054 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x81 binding to port GUID 0x200000
Jun 17 16:47:41 419878 [50493740] 0x02 -> osm_sm_bind: Bind to port guid 0x200000, port index 0 as main SM port
Jun 17 16:47:41 420017 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x03 binding to port GUID 0x200000
Jun 17 16:47:41 428618 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x04 binding to port GUID 0x200000
Jun 17 16:47:41 428727 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x21 binding to port GUID 0x200000
Jun 17 16:47:41 428769 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0a binding to port GUID 0x200000
Jun 17 16:47:41 428804 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0c binding to port GUID 0x200000
Jun 17 16:47:41 428838 [50493740] 0x02 -> osm_opensm_bind: Setting IS_SM on port 0x0000000000200000
Jun 17 16:47:41 429531 [3DA006C0] 0x02 -> do_sweep:


******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 17 16:47:41 429964 [3DA006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 17 16:47:41 444761 [3DA006C0] 0x80 -> Entering MASTER state
AR Manager - Configuration cycle (number 1) completed successfully
Jun 17 16:47:41 466414 [3DA006C0] 0x02 -> osm_ucast_mgr_process: ar_updn tables configured on all switches
Jun 17 16:47:41 466781 [3DA006C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 17 16:47:41 489882 [3DA006C0] 0x02 -> SUBNET UP
```

从日志可以看到 SM 启动以后，开启了一次 “heavy sweep”，后面会解释这个sweep。

"Bind to port guid 0x200000, port index 0 as main SM port" 这说明 OpenSM 把自己"挂"在了 Switch1 port 0上。记住这一点：后面在 ibsim 的报文日志里，你会看到几乎所有的管理报文都从 Switch1 的 port 0 出发，因为 SM 就住在那里，发现整张网络的旅程，正是从它脚下这台交换机开始的。

此时，ibsim 日志如下：

```bash
sim> ibwarn: [564977] sim_ctl_new_client: Attaching client 0 at default node "Switch1" port 0x200000
ibwarn: [564977] sim_ctl_set_issm: set issm 1 port 200000
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000002) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10000) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000005) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x30000) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x30001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000006) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000007) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000008) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000002) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10000) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000005) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000006) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x1) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x40000) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x40001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000007) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000008) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x1) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x19 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x19 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10000) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30000) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x1) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x1) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x101) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x101) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x201) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x201) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x301) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x301) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x401) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x401) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x501) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x501) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x601) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x601) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x701) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x701) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x801) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x801) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x3) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x4) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x103) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x104) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x203) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x204) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x303) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x304) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x403) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x404) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x503) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x504) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x603) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x604) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x703) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x704) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x803) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x804) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4


# ibsim 不断的周期性的有日志产生，这是 SM 在做 light sweep，后面会解释
sim>
sim>
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

sim>
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

sim>
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# 为了防止不断有新的日志打扰我们，暂时关闭日志功能
sim> verbose 0
simulator verbose level is 0

# 查看当前检测到的连接到 fabric 的设备，可以看到 SM
sim> attached
Client 0: pid 564980 connected at "Switch1" port 0x200000, lid 1, qp 0 SM

# 再次 dump，可以看到所有节点（包括Switch和HCA）都已经拿到了 lid，并且也可以看到每个交换机的 forwarding table
sim> dump
# Net status - Wed Jun 17 16:55:45 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]3 [4]FF [5]3
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Active/LinkUp
200000  [2]                       4x  2.5G Down/Polling
200000  [3]     "Switch2"[4]      4x  2.5G Active/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "Hca2"[1]         4x  2.5G Active/LinkUp
200001  [2]                       4x  2.5G Down/Polling
200001  [3]                       4x  2.5G Down/Polling
200001  [4]     "Switch1"[3]      4x  2.5G Active/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Switch2"[1]     lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 4 nodes

```

## 11.3 初始化的几个阶段

在钻进日志之前，先建立一张全景地图。一张 IB fabric 从上电到可用，SM 依次走过这几个阶段：

物理 fabric 建立（连好线）→ 子网发现（SM 探索拓扑）→ 信息收集（问清每个节点和端口的情况）→ LID 分配（给每个端口编号）→ 转发表计算与下发（算出路由、写进交换机）→ 端口配置（设置 LID、MTU、速率等参数）→ 子网激活（端口状态推进到 ACTIVE，网络可用）。

接下来我们对着真实日志，把这些阶段逐个看一遍。

## 11.4 SM 启动：从 DISCOVERING 到 SUBNET UP

OpenSM 的启动日志关键的几行是：

```bash
Jun 17 16:47:41 406790 [50493740] 0x80 -> Entering DISCOVERING state
Jun 17 16:47:41 408054 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x81 binding to port GUID 0x200000
Jun 17 16:47:41 419878 [50493740] 0x02 -> osm_sm_bind: Bind to port guid 0x200000, port index 0 as main SM port
Jun 17 16:47:41 420017 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x03 binding to port GUID 0x200000
Jun 17 16:47:41 428618 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x04 binding to port GUID 0x200000
Jun 17 16:47:41 428727 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x21 binding to port GUID 0x200000
Jun 17 16:47:41 428769 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0a binding to port GUID 0x200000
Jun 17 16:47:41 428804 [50493740] 0x02 -> osm_vendor_rebind: Mgmt class 0x0c binding to port GUID 0x200000
Jun 17 16:47:41 428838 [50493740] 0x02 -> osm_opensm_bind: Setting IS_SM on port 0x0000000000200000
Jun 17 16:47:41 429531 [3DA006C0] 0x02 -> do_sweep:

******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 17 16:47:41 429964 [3DA006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 0, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 17 16:47:41 444761 [3DA006C0] 0x80 -> Entering MASTER state
AR Manager - Configuration cycle (number 1) completed successfully
Jun 17 16:47:41 466414 [3DA006C0] 0x02 -> osm_ucast_mgr_process: ar_updn tables configured on all switches
Jun 17 16:47:41 466781 [3DA006C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 17 16:47:41 489882 [3DA006C0] 0x02 -> SUBNET UP
```

逐段翻译这条主线：

**进入 DISCOVERING 状态**：SM 一启动，先进入"发现"状态，准备探索网络。

**绑定各管理类**：`Mgmt class 0x81` 是子网管理接口（SMI，SM 用来发管理报文的通道），`0x03` 是子网管理（SA），`0x04` 是性能管理等。SM 启动时把这些管理服务的通道一一绑定到自己的端口上——这些正是 IB 管理面的几条标准通道。

**Setting IS_SM**：SM 在自己所在的端口上打出"我是 SM"的标记，向 fabric 宣告身份。

**HEAVY SWEEP START**：SM 发起第一次"重扫描"（heavy sweep）——一次对整张 fabric 的全量发现。注意这里的标志 `force_heavy_sweep 0`，表示这是启动时的常规全量扫描；后面我们会看到拓扑变化触发的扫描，那个标志是 1，对比起来很有意思。

**Entering MASTER state**：扫描完成，这个 SM 成为主 SM（master）。一个子网里可以有多个 SM，但只有一个是 master，其余待命。

**ucast_mgr_process**：单播路由管理器开始工作，为所有交换机计算并下发转发表。

**SUBNET UP**：子网就绪，整张网络进入可用状态。

这条 `DISCOVERING → 绑定管理类 → 标记 IS_SM → HEAVY SWEEP → MASTER → 算路由 → SUBNET UP` 的链路，就是 OpenSM 启动的完整故事。下面几节，我们把其中"发现"和"算路由"两个最关键的阶段拆开细看。

## 11.5 子网发现：SM 的问答记录

子网发现阶段，SM 做的事可以一句话概括：从自己脚下的交换机出发，逐跳地向每一个能触及的设备发问，把整张网络的样貌问清楚。

在 ibsim 的报文日志里，这个"逐跳发问"的过程是直接可见的。每一行 `process_packet: packet (attr 0xNN ...) reached host X port Y`，就是 SM 发出的一个管理报文（SMP）到达了某个设备，相当于 SM 问了这个设备一个问题，设备随即回答（`sim_read_pkt: replying`）。

日志的脉络是这样的（节选，不必逐条细究）：

```
packet (attr 0x11 ...) reached host Switch1 port 0      ← 先摸清脚下的 Switch1
packet (attr 0x10 ...) reached host Switch1 port 0
packet (attr 0x12 ...) reached host Switch1 port 0
...
packet (attr 0x11 ...) reached host Hca1 port 1         ← 顺着 Switch1 的端口发现 Hca1
packet (attr 0x11 ...) reached host Switch2 port 4      ← 也发现了对面的 Switch2
...
packet (attr 0x11 ...) reached host Hca2 port 1         ← 再顺着 Switch2 发现 Hca2
```

可以清楚看到 SM 的探索顺序：先问清自己所在的 Switch1，再顺着 Switch1 的活动端口发现直连的 Hca1 和对面的 Switch2，最后顺着 Switch2 发现挂在它后面的 Hca2。一圈下来，整张拓扑就被摸清了。

那些 `attr 0xNN` 是 SMP 的属性号，代表 SM 在问什么。标准的 SMP 属性号对照如下，带着它回看上面的日志，就能读懂 SM 每一步在打听什么：

| 属性号 | 名称                     | SM 在问什么                                             |
| ------ | ------------------------ | ------------------------------------------------------- |
| 0x10   | NodeDescription          | 你叫什么名字                                            |
| 0x11   | NodeInfo                 | 你是什么设备（交换机还是 HCA）、有几个端口、GUID 是多少 |
| 0x12   | SwitchInfo               | （交换机）你的转发表容量等交换参数                      |
| 0x14   | PortInfo                 | 这个端口的状态、LID、MTU、速率等                        |
| 0x15   | PartitionTable (P_Key)   | 这个端口的分区配置                                      |
| 0x16   | SLtoVLMappingTable       | SL 到 VL 的映射                                         |
| 0x17   | VLArbitrationTable       | VL 仲裁（QoS 调度）参数                                 |
| 0x18   | LinearForwardingTable    | 线性转发表（LFT）                                       |
| 0x19   | MulticastForwardingTable | 组播转发表                                              |

（这是 IBTA 标准 SMP 属性的参考定义。实际日志出自特定版本的 OpenSM + 模拟器，个别属性号的出现时机和次数未必与标准逐一对应，这里我们关注的是发现过程的整体脉络，不必纠结单条报文的精确含义。）

这里有一个值得停下来想一想的细节：SM 是怎么在还没有给任何设备分配 LID 的情况下，就能把报文发到这些设备的？

答案是 **directed routing（定向路由）**——发现阶段的 SMP 不靠 LID 寻址，而是按"从我出发，先走第几个端口，再走第几个端口"这样的逐跳路径来投递。正因如此，SM 才能在"网络里还没有任何地址"的最初时刻完成发现。这正是先前提到的"先有鸡还是先有蛋"问题的实际解法：发现用定向路由，地址是发现之后才分配的。（discover 行为与 DFS/BFS 非常类似）

> 从主机视角看，`ibnetdiscover` 把 SM 发现到的整张拓扑以可读的形式列出来，可以和上面日志里 SM 的探索过程相互印证。

再次新开一个terminal：

```bash
# 一定要先export，否则不会走 libumad2sim ，后面的测试会失败
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so

expert@net21:~$ ibnetdiscover
ibwarn: [565044] sim_connect: attached as client 1 at node "Switch1"
#
# Topology file: generated on Wed Jun 17 17:06:56 2026
#
# Initiated from node 0000000000200000 port 0000000000200000

vendid=0x0
devid=0x0
sysimgguid=0x200001
switchguid=0x200001(200001)
Switch  8 "S-0000000000200001"          # "Switch2" base port 0 lid 3 lmc 0
[1]     "H-0000000000100003"[1](100004)                 # "Hca2" lid 5 4xSDR
[4]     "S-0000000000200000"[3]         # "Switch1" lid 1 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x200000
switchguid=0x200000(200000)
Switch  8 "S-0000000000200000"          # "Switch1" base port 0 lid 1 lmc 0
[1]     "H-0000000000100000"[1](100001)                 # "Hca1" lid 2 4xSDR
[3]     "S-0000000000200001"[4]         # "Switch2" lid 3 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x100003
caguid=0x100003
Ca      2 "H-0000000000100003"          # "Hca2"
[1](100004)     "S-0000000000200001"[1]         # lid 5 lmc 0 "Switch2" lid 3 4xSDR

vendid=0x0
devid=0x0
sysimgguid=0x100000
caguid=0x100000
Ca      2 "H-0000000000100000"          # "Hca1"
[1](100001)     "S-0000000000200000"[1]         # lid 2 lmc 0 "Switch1" lid 1 4xSDR

```

## 11.6 LID 分配

发现完成、信息收集齐全之后，SM 给每个端口分配 LID。我们用模拟器的 `dump` 命令把结果看出来：

```bash
sim> dump
# Net status - Wed Jun 17 16:55:45 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]3 [4]FF [5]3
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Active/LinkUp
200000  [2]                       4x  2.5G Down/Polling
200000  [3]     "Switch2"[4]      4x  2.5G Active/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "Hca2"[1]         4x  2.5G Active/LinkUp
200001  [2]                       4x  2.5G Down/Polling
200001  [3]                       4x  2.5G Down/Polling
200001  [4]     "Switch1"[3]      4x  2.5G Active/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Switch2"[1]     lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 4 nodes
```

分配结果对照拓扑：Switch1 = lid 1、Hca1 = lid 2、Switch2 = lid 3、Hca2 = lid 5。

几个值得注意的点：

交换机的 LID 分配在 **port 0**（标为 "Sma Port"）。这是交换机的管理端口：交换机的众多数据端口本身不需要各自的 LID，整台交换机作为一个可管理实体，用 port 0 的 LID 来标识。HCA 则是每个工作端口都有自己的 LID。

每行的 `smlid 1` 表示：这个端口知道 SM 位于 lid 1。全网所有端口的 smlid 都指向 1，意味着大家都认同 lid 1（也就是 Switch1）上的那个 SM 是主管。

`lmc 0` 是 LID Mask Control，简单理解为每个端口只分到一个 LID（lmc 用于给一个端口分配多个连续 LID 以支持多路径，这里取 0）。

还可以注意到 lid 4 被跳过了：LID 的分配并不保证连续，SM 按自己的算法分配，中间出现空号是正常的，不必深究。

> 用 `ibaddr` 可以单独查某个端口的 LID/GID，`smpquery nodeinfo <lid>` 可以查到对应节点的 GUID、设备类型等，验证 dump 里看到的分配结果。

```bash
expert@net21:~$ ibaddr
ibwarn: [565051] sim_connect: attached as client 1 at node "Switch1"
GID fe80::20:0 LID start 0x1 end 0x1


expert@net21:~$ smpquery nodeinfo 5
ibwarn: [565054] sim_connect: attached as client 1 at node "Switch1"
# Node info: Lid 5
BaseVers:........................1
ClassVers:.......................1
NodeType:........................Channel Adapter
NumPorts:........................2
SystemGuid:......................0x0000000000100003
Guid:............................0x0000000000100003
PortGuid:........................0x0000000000100004
PartCap:.........................64
DevId:...........................0x0000
Revision:........................0x000000a1
LocalPort:.......................1
VendorId:........................0x000000


expert@net21:~$ smpquery nodeinfo 3
ibwarn: [565056] sim_connect: attached as client 1 at node "Switch1"
# Node info: Lid 3
BaseVers:........................1
ClassVers:.......................1
NodeType:........................Switch
NumPorts:........................8
SystemGuid:......................0x0000000000200001
Guid:............................0x0000000000200001
PortGuid:........................0x0000000000200001
PartCap:.........................8
DevId:...........................0x0000
Revision:........................0x000000a1
LocalPort:.......................0
VendorId:........................0x000000

```

## 11.7 转发表：LFT 如何工作

LID 分配好之后，SM 为每台交换机计算转发表（LFT），决定"发往某个 LID 的包，应该从本交换机的哪个端口送出去"。

直接看 dump 里 Switch1 的转发表：

```
Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]3 [4]FF [5]3
```

这一行的格式是 `[目的LID] 出端口`。逐项翻译，对照拓扑就完全清楚了：

| 目的 LID | 出端口 | 含义                                                   |
| -------- | ------ | ------------------------------------------------------ |
| [0]      | FF     | LID 0 无效，0xFF 表示无对应出端口                      |
| [1]      | 0      | 发给 lid 1（Switch1 自己）→ 走 port 0（本地管理端口）  |
| [2]      | 1      | 发给 lid 2（Hca1，直连在 port1）→ 从 port 1 送出，直达 |
| [3]      | 3      | 发给 lid 3（Switch2）→ 从 port 3（互联口）送出         |
| [4]      | FF     | LID 4 不存在，无路由                                   |
| [5]      | 3      | 发给 lid 5（Hca2）→ 也从 port 3 送出                   |

注意最后一行：Hca2（lid 5）并不直接连在 Switch1 上，它挂在 Switch2 后面。所以 Switch1 的转发表对 lid 5 的处理是"从 port 3 送出去"：也就是先送到 Switch2，剩下的交给 Switch2 接力。

再看 Switch2 的转发表印证这一点：

```
Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
```

发给 lid 5（Hca2，直连在 Switch2 的 port1）→ 从 port 1 送出，直达；发给 lid 1、lid 2（Switch1 一侧）→ 都从 port 4（Switch2 这端的互联口）送回去。

**两张表合起来，就构成了完整的多跳转发**：一个从 Hca1 发往 Hca2 的包，Switch1 查表知道"去 lid 5 走 port3"，把它送到 Switch2；Switch2 查表知道"去 lid 5 走 port1"，把它交给 Hca2。每台交换机只管自己这一跳，靠各自的 LFT 接力，包就走到了终点。所以，路径不是交换机自己学出来的，而是 SM 算好、分段写进每台交换机的转发表里的。

> `ibroute <lid>` 把单台交换机的 LFT 以"LID → 出端口"的表格形式列出，比 dump 里那行紧凑的十六进制更直观。

```bash
expert@net21:~$ ibroute 1
ibwarn: [565063] sim_connect: attached as client 1 at node "Switch1"
Unicast lids [0x0-0x5] of switch Lid 1 guid 0x0000000000200000 (Switch1):
  Lid  Out   Destination
       Port     Info
0x0001 000 : (Switch portguid 0x0000000000200000: 'Switch1')
0x0002 001 : (Channel Adapter portguid 0x0000000000100001: 'Hca1')
0x0003 003 : (Switch portguid 0x0000000000200001: 'Switch2')
0x0005 003 : (Channel Adapter portguid 0x0000000000100004: 'Hca2')
4 valid lids dumped

expert@net21:~$ ibroute 3
ibwarn: [565065] sim_connect: attached as client 1 at node "Switch1"
Unicast lids [0x0-0x5] of switch Lid 3 guid 0x0000000000200001 (Switch2):
  Lid  Out   Destination
       Port     Info
0x0001 004 : (Switch portguid 0x0000000000200000: 'Switch1')
0x0002 004 : (Channel Adapter portguid 0x0000000000100001: 'Hca1')
0x0003 000 : (Switch portguid 0x0000000000200001: 'Switch2')
0x0005 001 : (Channel Adapter portguid 0x0000000000100004: 'Hca2')
4 valid lids dumped
```

可以在ibsim上查看某个lid到另外一个lid的转发路径：

```bash
# Route <from-lid> <to-lid>
sim> route 5 2
From node "Hca2" port 1 lid 5
[1] -> "Switch2"[1]
[4] -> "Switch1"[3]
[1] -> "Hca1"[1]
To node "Hca1" port 1 lid 2
```

## 11.8 端口配置与子网激活

路由算完、下发完，SM 还要把每个端口的运行参数配置好，并把端口推进到可用状态。

**端口配置**：dump 里每个活动端口都带着 `4x 2.5G Active/LinkUp` 这样的描述，这里的 `4x`（链路宽度）、`2.5G`（速率）以及前面看到的 LID、MTU 等，都是 SM 在端口配置阶段设定的参数。

> `smpquery portinfo <lid> <port>` 能展开看某个端口的全部参数，LID、LMC、MTU、链路宽度/速率、当前状态等，对应 SM 在端口配置阶段写入的内容。相当于 `show interface xx`

```bash
expert@net21:~$ smpquery portinfo 3 1
ibwarn: [565072] sim_connect: attached as client 1 at node "Switch1"
# Port info: Lid 3 port 1
Mkey:............................<not displayed>
GidPrefix:.......................0x0000000000000000
Lid:.............................0
SMLid:...........................0
CapMask:.........................0x0
DiagCode:........................0x0000
MkeyLeasePeriod:.................0
LocalPort:.......................0
LinkWidthEnabled:................4X
LinkWidthSupported:..............1X or 4X or 8X or 12X or 2X
LinkWidthActive:.................4X
LinkSpeedSupported:..............2.5 Gbps or 5.0 Gbps or 10.0 Gbps
LinkState:.......................Active
PhysLinkState:...................LinkUp
LinkDownDefState:................Polling
ProtectBits:.....................0
LMC:.............................0
LinkSpeedActive:.................2.5 Gbps
LinkSpeedEnabled:................2.5 Gbps
NeighborMTU:.....................2048
SMSL:............................0
VLCap:...........................VL0-7
InitType:........................0x00
VLHighLimit:.....................0
VLArbHighCap:....................8
VLArbLowCap:.....................8
InitReply:.......................0x00
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................16
OperVLs:.........................VL0-1
PartEnforceInb:..................0
PartEnforceOutb:.................0
FilterRawInb:....................0
FilterRawOutb:...................0
MkeyViolations:..................0
PkeyViolations:..................0
QkeyViolations:..................0
GuidCap:.........................0
ClientReregister:................0
McastPkeyTrapSuppressionEnabled:.0
SubnetTimeout:...................0
RespTimeVal:.....................0
LocalPhysErr:....................0
OverrunErr:......................0
MaxCreditHint:...................0
RoundTrip:.......................0
CapabilityMask2:.................0x0
LinkSpeedExtActive:..............No Extended Speed
LinkSpeedExtSupported:...........0
LinkSpeedExtEnabled:.............0
LinkSpeedExtActive2:.............No Extended Speed 2
LinkSpeedExtSupported2:..........0
LinkSpeedExtEnabled2:............0
```

**子网激活**：IB 端口要经过一个状态机才能真正收发数据，依次是 DOWN → INIT → ARMED → ACTIVE。dump 里那些 `Active/LinkUp` 就是走完整个状态机、进入可用状态的端口；而没有接线的端口（如 Switch1 的 port 2、4、5等）则停在 `Down/Polling`，因为物理上没有对端，自然上不去。只有所有该激活的端口都到达 ACTIVE，整张子网才算真正激活。

## 11.9 SM 是活的：light sweep 与 heavy sweep

到这里，初始化的故事讲完了。但 SM 的工作并没有结束，它不是配置完就退场的一次性程序，而是持续运行、不断守护着这张网络。

SM 的运行时行为分两种扫描：

**light sweep（轻扫描）**：周期性地、轻量地巡查一遍，主要是检查各端口的状态有没有变化。开销小，频繁进行。

**heavy sweep（重扫描）**：一旦发现拓扑发生了变化（或收到设备主动上报的变化通知），就触发一次全量的重新发现 + 重算路由。开销大，只在需要时进行。

启动时那次全量扫描就是 heavy sweep。下面我们用 ibsim 主动制造一次拓扑变化，看 SM 如何实时反应。

### 断链：unlink 两台交换机之间的连线

在 ibsim 里执行 `unlink "Switch1"[3]`，断开两台交换机之间的互联。

```bash
# 先打开日志
sim> verbose 1
simulator verbose level is 1
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# unlink级联链路
sim> unlink "Switch1"[3]
ibwarn: [564977] send_trap: routing failed: no route to dest lid 1
sim> ibwarn: [564977] process_packet: lid 1 got trap repress - dropping
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000002) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10000) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000005) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000006) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000007) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000008) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x19 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# 关闭日志
sim> verbose 0
simulator verbose level is 0
```

OpenSM 的日志立刻有反应：

```bash
Jun 17 17:21:28 922890 [4E8006C0] 0x01 -> log_trap_info: Received Generic Notice type:1 num:128 (Link state change) Producer:2 (Switch) from LID:1 TID:0x0000000000000000
Jun 17 17:21:28 923199 [4E8006C0] 0x02 -> SM class trap 128: Directed Path Dump of 0 hop path: Path = 0
Jun 17 17:21:28 923315 [4E8006C0] 0x02 -> log_notice: Reporting Generic Notice type:1 num:128 (Link state change) from LID:1 GID:fe80::20:0
Jun 17 17:21:28 923669 [3DA006C0] 0x02 -> do_sweep:


******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 17 17:21:28 924124 [3DA006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 1, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 17 17:21:28 927893 [4F2006C0] 0x02 -> osm_pi_rcv_process: Switch 0x200000 Switch1 port 3 changed state from ACTIVE to DOWN
Jun 17 17:21:28 932678 [3DA006C0] 0x02 -> log_notice: Reporting Generic Notice type:3 num:65 (GID out of service) from LID:1 GID:fe80::20:1
Jun 17 17:21:28 932843 [3DA006C0] 0x02 -> drop_mgr_remove_port: Removed port with GUID:0x0000000000200001 LID range [3, 3] of node:Switch2
Jun 17 17:21:28 933044 [3DA006C0] 0x02 -> log_notice: Reporting Generic Notice type:3 num:65 (GID out of service) from LID:1 GID:fe80::10:4
Jun 17 17:21:28 933162 [3DA006C0] 0x02 -> drop_mgr_remove_port: Removed port with GUID:0x0000000000100004 LID range [5, 5] of node:Hca2
AR Manager - Configuration cycle (number 2) completed successfully
Jun 17 17:21:28 949992 [3DA006C0] 0x02 -> osm_ucast_mgr_process: ar_updn tables configured on all switches
Jun 17 17:21:28 950028 [3DA006C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 17 17:21:28 954937 [3DA006C0] 0x02 -> SUBNET UP
```

跟着这条因果链走：

1. **收到 trap**：设备主动上报了一个"链路状态变化"通知（num:128）。这正是前面讲过的——SMA 在端口状态变化时主动通知 SM，而不是等着 SM 下次来问。
2. **触发 heavy sweep**：注意这次的标志是 `force_heavy_sweep 1`，和启动时的 0 不同——这是被拓扑变化强制触发的重扫描。
3. **端口状态变化**：`Switch1 port 3 changed state from ACTIVE to DOWN`，互联口断了。
4. **摘除失联端口**：`drop_mgr_remove_port` 把 Switch2（lid 3）和 Hca2（lid 5）从可达集合里移除——因为通往它们的唯一链路断了，它们现在不可达。伴随的 `GID out of service` 是在通告"这些地址下线了"。
5. **重算并就绪**：重新计算路由后再次 `SUBNET UP`。

断链后 dump 一下，验证转发表的变化：

```bash
sim> dump
# Net status - Wed Jun 17 17:24:10 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]FF [4]FF [5]FF
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Active/LinkUp
200000  [2]                       4x  2.5G Down/Polling
200000  [3]                       4x  2.5G Down/Polling
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 1
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "Hca2"[1]         4x  2.5G Active/LinkUp
200001  [2]                       4x  2.5G Down/Polling
200001  [3]                       4x  2.5G Down/Polling
200001  [4]                       4x  2.5G Down/Polling
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Switch2"[1]     lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 4 nodes
```

从 Switch1 的视角，对比断链前的 `[3]3 [5]3`，现在去往 Switch2（lid 3）和 Hca2（lid 5）的表项都变成了 `FF`，即不可达。SM 如实地把"这两个目的现在去不了"反映到了转发表里。Hca2 整个从这张网络的可达版图上消失了。

### 恢复：relink 重新接上

再执行 `relink "Switch1"[3]` 把线接回来，日志又是一轮反应：

```bash
# 开启日志
sim> verbose 1
simulator verbose level is 1
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# 重新开启级联
sim> relink "Switch1"[3]
ibwarn: [564977] send_trap: routing failed: no route to dest lid 1
sim> ibwarn: [564977] process_packet: lid 1 got trap repress - dropping
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000002) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10000) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000005) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x30000) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x30001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000006) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000007) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000008) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x1) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000002) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x11 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10000) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x10001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000005) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000006) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x1) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x40000) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x40001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000007) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000008) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x16 mod 0x1) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x19 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x19 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10000) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30003) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30000) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30000) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x3) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x10004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x103) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x18 mod 0x30004) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x1) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x203) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x303) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x101) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x201) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x403) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x503) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x301) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x401) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x603) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x703) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x501) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x601) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x803) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x701) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x801) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x4) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x104) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x204) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x304) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x404) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x504) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x604) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x704) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x17 mod 0x804) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x15 mod 0x80000001) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca1 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x10 mod 0x0) reached host Hca2 port 1
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# 关闭日志
sim> verbose 0
simulator verbose level is 0
```

OpenSM日志：

```bash
Jun 17 17:25:44 598054 [4E8006C0] 0x01 -> log_trap_info: Received Generic Notice type:1 num:128 (Link state change) Producer:2 (Switch) from LID:1 TID:0x0000000000000000
Jun 17 17:25:44 598342 [4E8006C0] 0x02 -> SM class trap 128: Directed Path Dump of 0 hop path: Path = 0
Jun 17 17:25:44 598396 [4E8006C0] 0x02 -> log_notice: Reporting Generic Notice type:1 num:128 (Link state change) from LID:1 GID:fe80::20:0
Jun 17 17:25:44 598661 [3DA006C0] 0x02 -> do_sweep:


******************************************************************
*********************** HEAVY SWEEP START ************************
******************************************************************


Jun 17 17:25:44 599109 [3DA006C0] 0x02 -> do_sweep: Entering heavy sweep with flags: force_heavy_sweep 1, coming out of standby 0, subnet initialization error 0, sm port change 0
Jun 17 17:25:44 603649 [4DE006C0] 0x02 -> osm_pi_rcv_process: Switch 0x200000 Switch1 port 3 changed state from DOWN to INIT
AR Manager - Configuration cycle (number 3) completed successfully
Jun 17 17:25:44 629689 [3DA006C0] 0x02 -> osm_ucast_mgr_process: ar_updn tables configured on all switches
Jun 17 17:25:44 629766 [3DA006C0] 0x02 -> osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
Jun 17 17:25:44 651781 [3DA006C0] 0x02 -> log_notice: Reporting Generic Notice type:3 num:64 (GID in service) from LID:1 GID:fe80::20:1
Jun 17 17:25:44 651828 [3DA006C0] 0x02 -> state_mgr_report_new_ports: Discovered new port with GUID:0x0000000000200001 LID range [3,3] of node: Switch2
Jun 17 17:25:44 651851 [3DA006C0] 0x02 -> log_notice: Reporting Generic Notice type:3 num:64 (GID in service) from LID:1 GID:fe80::10:4
Jun 17 17:25:44 651872 [3DA006C0] 0x02 -> state_mgr_report_new_ports: Discovered new port with GUID:0x0000000000100004 LID range [5,5] of node: Hca2
Jun 17 17:25:44 651905 [3DA006C0] 0x02 -> SUBNET UP
```

这次能看到端口状态机重新往上爬：`port 3 changed state from DOWN to INIT`——从断开状态进入初始化状态，开始重新走那条 DOWN → INIT → ... → ACTIVE 的激活路径。随后 `GID in service` 和 `Discovered new port ... Switch2 / Hca2` 表明 SM 重新发现了这两个端口，把它们重新纳入网络。

再 dump 验证，转发表恢复如初：

```bash
sim> dump
# Net status - Wed Jun 17 17:28:22 2026

Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]3 [4]FF [5]3
200000  [0]     "Sma Port"[0]    lid 1 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200000  [1]     "Hca1"[1]         4x  2.5G Active/LinkUp
200000  [2]                       4x  2.5G Down/Polling
200000  [3]     "Switch2"[4]      4x  2.5G Active/LinkUp
200000  [4]                       4x  2.5G Down/Polling
200000  [5]                       4x  2.5G Down/Polling
200000  [6]                       4x  2.5G Down/Polling
200000  [7]                       4x  2.5G Down/Polling
200000  [8]                       4x  2.5G Down/Polling

Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
200001  [0]     "Sma Port"[0]    lid 3 lmc 0 smlid 1  4x  2.5G Active/LinkUp
200001  [1]     "Hca2"[1]         4x  2.5G Active/LinkUp
200001  [2]                       4x  2.5G Down/Polling
200001  [3]                       4x  2.5G Down/Polling
200001  [4]     "Switch1"[3]      4x  2.5G Active/LinkUp
200001  [5]                       4x  2.5G Down/Polling
200001  [6]                       4x  2.5G Down/Polling
200001  [7]                       4x  2.5G Down/Polling
200001  [8]                       4x  2.5G Down/Polling

Ca 2 "Hca1"     nodeguid 100000 sysimgguid 100000
100001  [1]     "Switch1"[1]     lid 2 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100002  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling

Ca 2 "Hca2"     nodeguid 100003 sysimgguid 100003
100004  [1]     "Switch2"[1]     lid 5 lmc 0 smlid 1  4x  2.5G Active/LinkUp
100005  [2]                      lid 0 lmc 0 smlid 0  4x  2.5G Down/Polling
#  dumped 4 nodes
```

Switch1 去往 Switch2 和 Hca2 的路由又回来了，网络完全恢复。

所以，SM 持续监控着 fabric。任何拓扑变化都会被 trap 上报、触发 SM heavy sweep、重新发现并重算路由，最后 SM 把结果写进每台交换机的转发表。

## 11.10 小结

我们用 ibsim + opensm，把一张 IB fabric 从"插好线"到"能通信"的全过程看了一遍：

SM 启动后进入 DISCOVERING，用定向路由的 SMP 从脚下的交换机出发逐跳发现整张拓扑，收集每个节点和端口的信息，给每个端口分配 LID，为每台交换机计算并下发 LFT，配置端口参数并把它们推进到 ACTIVE，最终 SUBNET UP：一张每个端口有地址、每台交换机有转发表、任意两点可达的网络就此诞生。

而通过 unlink/relink 的演示，我们还看到 SM 并非配置完就退场：它持续运行，靠 light sweep 周期巡查、靠 heavy sweep 响应拓扑变化，始终维护着这张网络的正确状态。

OpenSM 就是那个一直在后台运转的循环：发现拓扑 → 编号 → 算路由 → 下发 → 持续监控。前面几章里那些抽象的概念（LID、LFT、SMP、SM）到此已经非常清晰。
