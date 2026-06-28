# Chapter 11: InfiniBand Fabric Initialization

## 11.1 Between "Cables Plugged In" and "Able to Communicate"

For IB, once all the cables are plugged in, the devices have physical connectivity—but this is not yet a usable network. At this point, every port in the IB Fabric still has no address, every switch still doesn't know where to forward packets, and the whole fabric is a pile of devices that are connected to each other yet don't recognize one another.

From this state to a network where every port has a LID, every switch has a forwarding table, and any two points can communicate, the thing doing the work in between is the Subnet Manager (SM). On real hardware, this process completes automatically within a second or two after you power on the devices, so fast you can't even see it. In this chapter, with the ibsim + opensm environment we set up earlier, we slow this process down and look step by step at exactly what the SM does.

The goal of this chapter is to lift the veil on OpenSM. You'll see that the SM is essentially a loop of "discover topology → number → compute routes → push down → continuously monitor."

## 11.2 The Demo Topology and Runtime Environment

All demos in this chapter are based on this topology: two switches interconnected, each with one HCA attached.

The ibsim topology file is as follows, stored in `~/ib1.net`:

```bash
#      Switch1 ──────── Switch2
#         |                |
#         |                |
#        Hca1             Hca2

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

Start ibsim in a terminal to launch the simulated fabric, and you'll see the following logs:

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

From the logs above, via dump, we can see that all devices' LIDs are 0, because at this point the SM is not yet online.

Next, open a new terminal for OpenSM. OpenSM hooks onto the simulated fabric through the `umad2sim` preload library provided by ibsim. After it starts, you can see at the beginning of the log:

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

From the log we can see that after the SM starts, it kicks off a "heavy sweep," which will be explained later.

"Bind to port guid 0x200000, port index 0 as main SM port" indicates that OpenSM has "attached" itself to Switch1 port 0. Remember this: later, in ibsim's packet logs, you'll see that almost all management packets set off from Switch1's port 0, because that's where the SM lives—the journey to discover the entire network begins precisely from the switch right beneath its feet.

At this point, the ibsim log is as follows:

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


# ibsim keeps producing logs periodically; this is the SM doing a light sweep, which will be explained later
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

# To prevent new logs from constantly interrupting us, temporarily turn off the logging
sim> verbose 0
simulator verbose level is 0

# View the devices currently detected as connected to the fabric; you can see the SM
sim> attached
Client 0: pid 564980 connected at "Switch1" port 0x200000, lid 1, qp 0 SM

# Dump again; you can see that all nodes (including Switches and HCAs) have obtained a LID, and you can also see each switch's forwarding table
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

## 11.3 The Stages of Initialization

Before diving into the logs, let's first build a panoramic map. From power-on to usable, an IB fabric goes through these stages in order with the SM:

Physical fabric established (cables connected) → subnet discovery (SM explores the topology) → information gathering (asking each node and port about its situation) → LID assignment (numbering each port) → forwarding table computation and push-down (computing routes, writing them into the switches) → port configuration (setting parameters such as LID, MTU, rate) → subnet activation (port states advance to ACTIVE, the network is usable).

Next, we'll look at these stages one by one against the real logs.

## 11.4 SM Startup: From DISCOVERING to SUBNET UP

The key lines of OpenSM's startup log are:

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

Translating this main thread segment by segment:

**Entering the DISCOVERING state**: as soon as the SM starts, it first enters the "discovering" state, preparing to explore the network.

**Binding the management classes**: `Mgmt class 0x81` is the Subnet Management Interface (SMI, the channel the SM uses to send management packets), `0x03` is Subnet Administration (SA), `0x04` is performance management, and so on. When the SM starts, it binds these management service channels one by one to its own port. These are several standard channels of the IB management plane, traveling over QP0 / QP1.

**Setting IS_SM**: the SM stamps the "I am the SM" mark on its own port, announcing its identity to the fabric.

**HEAVY SWEEP START**: the SM initiates its first "heavy sweep"—a full discovery of the entire fabric. Note the flag `force_heavy_sweep 0` here, indicating this is the regular full scan at startup; later we'll see scans triggered by topology changes, where that flag is 1—an interesting contrast.

**Entering MASTER state**: the scan completes, and this SM becomes the master SM. A subnet can have multiple SMs, but only one is the master, with the rest on standby.

**ucast_mgr_process**: the unicast routing manager starts working, computing and pushing down the forwarding tables for all switches.

**SUBNET UP**: the subnet is ready, and the entire network enters the usable state.

This chain of `DISCOVERING → bind management classes → mark IS_SM → HEAVY SWEEP → MASTER → compute routes → SUBNET UP` is the complete story of OpenSM's startup. In the next few sections, we'll break down the two most crucial stages, "discovery" and "computing routes," and look at them in detail.

## 11.5 Subnet Discovery: The SM's Q&A Record

The discovery stage's work can be summed up in one sentence: starting from the switch right beneath its feet, the SM hop by hop asks every device it can reach, getting the whole network's appearance clear.

In ibsim's packet log, this "hop-by-hop questioning" process is directly visible. Each line `process_packet: packet (attr 0xNN ...) reached host X port Y` is a management packet (SMP) sent by the SM arriving at some device, equivalent to the SM asking that device a question, and the device immediately answering (`sim_read_pkt: replying`).

The thread of the log is like this (excerpted; no need to scrutinize line by line):

```
packet (attr 0x11 ...) reached host Switch1 port 0      ← first get clear on the Switch1 right under its feet
packet (attr 0x10 ...) reached host Switch1 port 0
packet (attr 0x12 ...) reached host Switch1 port 0
...
packet (attr 0x11 ...) reached host Hca1 port 1         ← following Switch1's port, discover Hca1
packet (attr 0x11 ...) reached host Switch2 port 4      ← also discovered the opposite Switch2
...
packet (attr 0x11 ...) reached host Hca2 port 1         ← then following Switch2, discover Hca2
```

You can clearly see the SM's exploration order: first get the Switch1 it's on clear, then follow Switch1's active ports to discover the directly connected Hca1 and the opposite Switch2, and finally follow Switch2 to discover Hca2 hanging behind it. After one round, the whole topology is mapped out.

Those `attr 0xNN` are the SMP attribute numbers, representing what the SM is asking. The standard SMP attribute number reference is as follows; with it, looking back at the logs above, you can read what the SM is inquiring about at each step:

| Attribute number | Name                     | What the SM is asking                                    |
| ---------------- | ------------------------ | ------------------------------------------------------- |
| 0x10             | NodeDescription          | What is your name                                       |
| 0x11             | NodeInfo                 | What device are you (switch or HCA), how many ports, what's your GUID |
| 0x12             | SwitchInfo               | (switch) your forwarding table capacity and other switching parameters |
| 0x14             | PortInfo                 | This port's state, LID, MTU, rate, etc.                 |
| 0x15             | PartitionTable (P_Key)   | This port's partition configuration                     |
| 0x16             | SLtoVLMappingTable       | The SL-to-VL mapping                                    |
| 0x17             | VLArbitrationTable       | VL arbitration (QoS scheduling) parameters             |
| 0x18             | LinearForwardingTable    | The Linear Forwarding Table (LFT)                       |
| 0x19             | MulticastForwardingTable | The multicast forwarding table                          |

(This is a reference definition of the IBTA standard SMP attributes. The actual logs come from a specific version of OpenSM + the simulator, and the timing and frequency of individual attribute numbers may not correspond one-to-one with the standard. Here we focus on the overall thread of the discovery process and need not fuss over the precise meaning of a single packet.)

There's a detail here worth pausing to think about: how is the SM able to send packets to these devices when it hasn't yet assigned a LID to any device?

The answer is **directed routing**. The SMPs in the discovery stage don't address by LID but are delivered along a hop-by-hop path of "from me, first go out which port, then go out which port." Precisely because of this, the SM can complete discovery at the very first moment when "there are no addresses in the network yet." (The discovery behavior is very similar to a DFS/BFS algorithm.)

> From the host's perspective, `ibnetdiscover` lists the entire topology the SM discovered in readable form, which can be cross-checked against the SM's exploration process in the logs above.

Open another new terminal:

```bash
# Be sure to export first, otherwise it won't go through libumad2sim, and the subsequent tests will fail
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

## 11.6 LID Assignment

After discovery completes and the information is fully gathered, the SM assigns a LID to each port. Let's see the result with the simulator's `dump` command:

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

The assignment result against the topology: Switch1 = lid 1, Hca1 = lid 2, Switch2 = lid 3, Hca2 = lid 5.

A few points worth noting:

- A switch's LID is assigned to **port 0** (labeled "Sma Port"). This is the switch's management port: the switch's many data ports don't need their own individual LIDs; the entire switch, as one manageable entity, is identified by port 0's LID. An HCA, on the other hand, has its own LID for each working port.

- The `smlid 1` on each line indicates that the master SM managing it is at LID 1. This value is written by the master SM into each port's PortInfo during the configuration scan.

- `lmc 0` is the LID Mask Control, used to assign multiple consecutive LIDs to a port to support multi-pathing. A value of 0 here can simply be understood as each port getting only one LID.

- You may also notice that lid 4 is skipped, because LID assignment is not guaranteed to be contiguous; the SM assigns by its own algorithm, and gaps in between are normal and need not be dwelt on.

---

With `ibaddr` you can query a particular port's LID/GID individually, and `smpquery nodeinfo <lid>` can query the corresponding node's GUID, device type, etc., verifying the assignment results seen in dump.

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

## 11.7 The Forwarding Table: How the LFT Works

After the LIDs are assigned, the SM computes the forwarding table (LFT) for each switch, deciding "for a packet destined to a certain LID, which port of this switch should it be sent out of."

Let's look directly at Switch1's forwarding table in the dump:

```bash
Switch 8 "Switch1"      nodeguid 200000 sysimgguid 200000
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]0 [2]1 [3]3 [4]FF [5]3
```

The format of this line is `[destination LID] egress port`. Translating item by item, against the topology, it's completely clear:

| Destination LID | Egress port | Meaning                                                |
| --------------- | ----------- | ------------------------------------------------------ |
| [0]             | FF          | LID 0 is invalid; 0xFF means no corresponding egress port |
| [1]             | 0           | sent to lid 1 (Switch1 itself) → goes out port 0 (the local management port) |
| [2]             | 1           | sent to lid 2 (Hca1, directly connected to port1) → out port 1, direct |
| [3]             | 3           | sent to lid 3 (Switch2) → out port 3 (the interconnect port) |
| [4]             | FF          | LID 4 doesn't exist, no route                          |
| [5]             | 3           | sent to lid 5 (Hca2) → also out port 3                  |

Note the last line: Hca2 (lid 5) is not directly connected to Switch1; it hangs behind Switch2. So Switch1's forwarding table handles lid 5 as "send out port 3": that is, first send it to Switch2 and leave the rest to Switch2 to relay.

Now look at Switch2's forwarding table to confirm this:

```bash
Switch 8 "Switch2"      nodeguid 200001 sysimgguid 200001
#       linearcap 49152 FDBtop 5 portchange 0
#       Forwarding table 0-5: [0]FF [1]4 [2]4 [3]0 [4]FF [5]1
```

Sent to lid 5 (Hca2, directly connected to Switch2's port1) → out port 1, direct; sent to lid 1, lid 2 (the Switch1 side) → both go back out port 4 (Switch2's interconnect port on this end).

**The two tables together constitute complete multi-hop forwarding**: a packet sent from Hca1 to Hca2—Switch1 looks up the table and knows "to lid 5, go out port3," sending it to Switch2; Switch2 looks up the table and knows "to lid 5, go out port1," handing it to Hca2. Each switch handles only its own hop, relaying via its respective LFT, and the packet reaches its destination. So the path is not learned by the switches themselves, but computed by the SM and written, segment by segment, into each switch's forwarding table.

---

`ibroute <lid>` lists a single switch's LFT in a "LID → egress port" table form, more intuitive than that compact hexadecimal line in the dump.

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

You can also view the forwarding path from one lid to another on ibsim:

```bash
# Route <from-lid> <to-lid>
sim> route 5 2
From node "Hca2" port 1 lid 5
[1] -> "Switch2"[1]
[4] -> "Switch1"[3]
[1] -> "Hca1"[1]
To node "Hca1" port 1 lid 2
```

## 11.8 Port Configuration and Subnet Activation

After routes are computed and pushed down, the SM still has to configure each port's runtime parameters and advance the ports to a usable state.

**Port configuration**: in the dump, each active port carries a description like `4x 2.5G Active/LinkUp`. Here the `4x` (link width), `2.5G` (rate), and the LID, MTU, etc. seen earlier are all parameters set by the SM during the port configuration stage.

> `smpquery portinfo <lid> <port>` can expand and show all the parameters of a port—LID, LMC, MTU, link width/rate, current state, etc.—corresponding to what the SM wrote during the port configuration stage. It's the equivalent of the `show interface xx` that network engineers are familiar with.

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

**Subnet activation**: an IB port must go through a state machine before it can truly send and receive data, in order: DOWN → INIT → ARMED → ACTIVE. Those `Active/LinkUp` in the dump are ports that have walked through the whole state machine and entered the usable state; while ports without a cable (such as Switch1's port 2, 4, 5, etc.) stay at `Down/Polling`, because there's no peer physically, so naturally they can't come up. Only when all ports that should be activated reach ACTIVE is the entire subnet truly activated.

## 11.9 The SM Is Alive: light sweep and heavy sweep

By now, the story of initialization is told. But the SM's work is not over; it is not a one-time program that exits after configuration, but runs continuously, constantly guarding this network.

The SM has two scan modes:

**light sweep**: periodically and lightly patrols once, mainly checking whether the state of each port has changed. Low overhead, performed frequently.

**heavy sweep**: once a topology change is found (or a change notification proactively reported by a device is received), it triggers a full re-discovery + route recomputation. High overhead, performed only when needed.

That full scan at startup was a heavy sweep. Below, we use ibsim to proactively create a topology change and see how the SM reacts in real time.

### Breaking the Link: unlink the Connection Between the Two Switches

In ibsim, execute `unlink "Switch1"[3]` to break the interconnect between the two switches.

```bash
# Turn on the log first
sim> verbose 1
simulator verbose level is 1
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4
ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch2 port 4
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# unlink the cascade link
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

# Turn off the log
sim> verbose 0
simulator verbose level is 0
```

OpenSM's log reacts immediately:

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

Following this causal chain:

1. **Receive a trap**: a device proactively reported a "link state change" notification (num:128). This shows that an SMA proactively notifies the SM when a port state changes, rather than waiting for the SM to come ask next time.
2. **Trigger a heavy sweep**: note that this time the flag is `force_heavy_sweep 1`, different from the 0 at startup—this is a heavy sweep forcibly triggered by the topology change.
3. **Port state change**: `Switch1 port 3 changed state from ACTIVE to DOWN`, the interconnect port is broken.
4. **Remove the unreachable ports**: `drop_mgr_remove_port` removes Switch2 (lid 3) and Hca2 (lid 5) from the reachable set—because the only link to them is broken, they are now unreachable. The accompanying `GID out of service` is announcing "these addresses have gone offline."
5. **Recompute and become ready**: after recomputing routes, `SUBNET UP` again.

Dump after breaking the link to verify the change in the forwarding table:

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

From Switch1's perspective, compared with `[3]3 [5]3` before the link broke, the entries to Switch2 (lid 3) and Hca2 (lid 5) have now both become `FF`, i.e., unreachable. The SM faithfully reflects "these two destinations can't be reached now" into the forwarding table. Hca2 has vanished entirely from this network's reachable map.

### Recovery: relink to Reconnect

Execute `relink "Switch1"[3]` to reconnect the cable, and the log goes through another round of reaction:

```bash
# Turn on the log
sim> verbose 1
simulator verbose level is 1
sim> ibwarn: [564977] process_packet: packet (attr 0x12 mod 0x0) reached host Switch1 port 0
ibwarn: [564977] sim_read_pkt: replying 288 bytes (288) to client 0 fd 4

# Re-enable the cascade
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

# Turn off the log
sim> verbose 0
simulator verbose level is 0
```

OpenSM log:

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

This time you can see the port state machine climbing back up: `port 3 changed state from DOWN to INIT`—from the broken state into the initialization state, starting to walk that DOWN → INIT → ... → ACTIVE activation path again. The subsequent `GID in service` and `Discovered new port ... Switch2 / Hca2` indicate that the SM has rediscovered these two ports and brought them back into the network.

Dump again to verify; the forwarding table has been restored to its original state:

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

Switch1's routes to Switch2 and Hca2 are back, and the network is fully restored.

So, the SM continuously monitors the fabric. Any topology change is reported by a trap, triggers an SM heavy sweep, re-discovers and recomputes routes, and finally the SM writes the result into each switch's forwarding table.

## 11.10 Summary

Using ibsim + opensm, we've watched the entire process of an IB fabric going from "cables plugged in" to "able to communicate":

After the SM starts, it enters DISCOVERING, uses directed-routing SMPs to discover the entire topology hop by hop starting from the switch beneath its feet, gathers information about each node and port, assigns a LID to each port, computes and pushes down the LFT for each switch, configures port parameters and advances them to ACTIVE, and finally reaches SUBNET UP. A network where every port has an address, every switch has a forwarding table, and any two points are reachable is thus born.

Through the unlink/relink demo, we also saw that the SM doesn't exit after configuration: it runs continuously, patrolling periodically with light sweeps and responding to topology changes with heavy sweeps, always maintaining the correct state of this network.

OpenSM's working logic can be summarized as: discover topology → number → compute routes → push down → continuously monitor.
