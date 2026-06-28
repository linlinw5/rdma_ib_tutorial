# Chapter 9: Setting Up the InfiniBand Lab Environment

## 9.1 Overview

IB hardware has too low an adoption rate, so there is naturally a barrier to studying it. I don't have any IB switches or NICs on hand either, so to keep exploring InfiniBand I can only find a way to rely on software simulation.

This chapter introduces the process of building a complete IB lab environment using **ibsim** (a fabric simulator) + **opensm** (the Subnet Manager), without any real IB hardware.

**The version combination verified to work:**

| Component    | Version              |
| ------------ | -------------------- |
| OS           | Ubuntu 24.04         |
| opensm       | 5.21.12.MLNX20250617 |
| ibsim        | 0.12                 |
| Software repo | NVIDIA DOCA 2.9.4   |

This chapter focuses mainly on environment setup, briefly listing the basic test commands; later chapters will cover them in detail.

---

## 9.2 Introduction to ibsim and OpenSM

**ibsim**

ibsim is an InfiniBand subnet simulator whose core role is to virtualize a complete IB subnet topology on a single Linux host, without any physical HCA hardware. It exposes its interface externally by creating virtual umad (user-space MAD) devices, so that all standard IB management tools (including OpenSM, ibnetdiscover, ibtracert, etc.) can interact with it as if they were operating real hardware. ibsim reads node and link definitions from a topology description file, and once started it maintains a simulated MAD response table that can respond to attribute queries in SMPs (Subnet Management Packets) such as NodeInfo, PortInfo, SwitchInfo, and LinearForwardingTable, and can also accept LID assignments and LFT write operations pushed down by OpenSM.

For development, testing, and teaching scenarios, ibsim's greatest value is that it completely decouples "topology scale" from "hardware cost." We can simulate a fat-tree network containing dozens of switches and hundreds of ports on a single laptop, observe how OpenSM discovers the topology, computes routes, and assigns LIDs, and trace the round-trip path of every Directed Route SMP by capturing umad packets—all without a single real Mellanox HCA.

**OpenSM**

OpenSM is the open-source implementation of the Subnet Manager (SM) defined by the InfiniBand specification, and is a necessary condition for an IB subnet to work properly. In the IB architecture, the SM is the "control-plane brain" of the entire subnet: it is responsible for topology discovery (traversing all nodes via Directed Route SMPs), LID assignment, route computation (pushing down LFTs/MFTs), and advancing the subnet's state machine from Discovering to Armed to Active.

OpenSM runs as a user-space process, sending and receiving MAD packets through the /dev/infiniband/umad\* character devices.

In terms of deployment, OpenSM has considerable flexibility: in our lab environment, it runs directly on an ordinary Linux host, working with ibsim to manage the simulated subnet; whereas in production, OpenSM is usually embedded and runs inside the IB switch firmware—Mellanox/NVIDIA IB switches carry the SM function this way, completing autonomous subnet management without any external host involvement.

**The Implementation Principle of the Simulated Environment**

Our lab environment is just one Ubuntu Linux machine, with ibsim and OpenSM installed on it. The principle of communication between them is as follows:

```
opensm / diagnostic tools
      ↓
  libibumad          ← user-space MAD access library (LD = Linker/Loader)
      ↓
  LD_PRELOAD=libumad2sim.so   ← intercepts umad calls, redirects to ibsim
      ↓
  ibsim              ← simulates the IB fabric's MAD responses
```

```bash
# `libumad2sim.so` redefines the same-named functions
libibumad provides:    umad_open()  umad_send()  umad_recv() ...
libumad2sim also provides: umad_open()  umad_send()  umad_recv() ... (same names)
```

By setting an environment variable (`export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so`), when the dynamic linker resolves the function symbols it first finds the versions in `libumad2sim`, and the same-named functions in `libibumad` are **shadowed**.

Inside `libumad2sim`'s implementation, the behavior of these functions is changed from "read/write `/dev/infiniband/umadX`" to "read/write ibsim's unix socket."

Thereby the experiment redirects the MAD packets of OpenSM and the diagnostic tools to ibsim's unix socket, with no real hardware needed and no recompilation of any program needed.

---

## 9.3 Installation

OpenSM and ibsim are both open-source software themselves; on a clean Ubuntu system you can just use `sudo apt install`.

For convenience, and to demonstrate how to install MLNX_OFED, next we use the NVIDIA DOCA apt repo for a one-command install.

URL: `https://linux.mellanox.com/public/repo/doca`

doca-ofed is a meta-package that automatically pulls in all dependencies (opensm, ibsim, infiniband-diags, libibmad5, libibumad3, etc.)

> Version note: DOCA 2.9.4 corresponds to opensm 5.21.12, which is compatible with ibsim 0.12.

> During testing, I found that DOCA 3.3.0 corresponds to opensm 5.26.x, which has compatibility issues with ibsim 0.12—the SM gets stuck in the DISCOVERING phase.

```bash
# Add the NVIDIA DOCA 2.9.4 repo
curl https://linux.mellanox.com/public/repo/doca/GPG-KEY-Mellanox.pub \
  | gpg --dearmor \
  | sudo tee /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub > /dev/null

echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub] \
  https://linux.mellanox.com/public/repo/doca/2.9.4/ubuntu24.04/x86_64/ ./" \
  | sudo tee /etc/apt/sources.list.d/doca.list

sudo apt-get update

# One-command install (handles all dependencies automatically)
sudo apt-get install doca-ofed -y

# rlwrap: provides backspace and history support for the ibsim console
sudo apt-get install rlwrap -y
```

After installation, all the following commands are available:

| Command                                         | Source package   |
| ----------------------------------------------- | ---------------- |
| `opensm`                                        | opensm           |
| `ibsim`                                         | ibsim            |
| `ibnetdiscover`, `ibnodes`, `ibswitches`        | infiniband-diags |
| `smpquery`, `saquery`, `dump_lfts`, `ibtracert` | infiniband-diags |

---

## 9.4 The Topology File

ibsim uses a text file to describe the fabric topology, in a format compatible with the output of `ibnetdiscover`.

### Format Specification

```
# Node header
type(Switch|Hca)  port_count  "nodeid"

# Port connection
[localport]  "remoteid"[remoteport]
```

### Example: Dual-Switch Topology (net-examples/net)

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

> For first-time use, it's recommended to just use the example topology file that comes with ibsim:
> `/usr/share/doc/ibsim/net-examples/net`

Port 0 of each Switch is the **SMA Port** (Subnet Management Agent), a management logical port built into the switch that is always in the Active state and is responsible for responding to the SMP MADs sent by the SM. It is not a physical port and has no physical link.

---

## 9.5 Startup Procedure

Starting OpenSM and ibsim requires three separate terminal windows, used respectively for: the ibsim console, the OpenSM console, and running management tools.

### Terminal 1: Start ibsim

```bash
# rlwrap provides readline features such as backspace and scrolling through history (ibsim does not natively support them)
rlwrap ibsim -s -v /usr/share/doc/ibsim/net-examples/net
```

Parameter explanation:

| Parameter        | Description                                       |
| ---------------- | ------------------------------------------------- |
| `-s`             | Start the network immediately, without waiting for the console `Start` command |
| `-v`             | verbose mode, prints details of MAD sends/receives |
| The last parameter | Path to the topology file                       |

After a successful start it displays:

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

### Verify the Topology

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

Confirm that all nodes and connections are correct, with ports showing `Init/LinkUp`.

### Terminal 2: Start opensm

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -f -'
```

Parameter explanation:

| Parameter            | Description                                                                  |
| -------------------- | --------------------------------------------------------------------------- |
| `sudo bash -c '...'` | Set the environment variable inside a sudo sub-shell, ensuring `LD_PRELOAD` is not cleared by sudo |
| `LD_PRELOAD=...`     | Preload `libumad2sim.so`, redirecting umad calls to ibsim                    |
| `-q local`           | Enable the local console for OpenSM (optional)                              |
| `-f -`               | Output logs to stdout (`-` means standard output); logs can also be output to a file, see the help docs for details |

Note: OpenSM's console isn't very useful in this experiment, so the `-q` parameter can also be omitted.

After the SM converges normally, opensm outputs:

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

### Verify the Change in ibsim

After opensm starts, use the `Attached` command in the ibsim console to confirm the connection:

```bash
# Turn off log printing
sim> verbose 0
simulator verbose level is 0

sim> attached
Client 0: pid 6877 connected at "Switch1" port 0x200000, lid 1, qp 0 SM
# opensm has successfully connected, attached to Switch1's management port (port 0, GUID 0x200000), and was assigned LID 1
```

Looking at ibsim's `Dump` again, it should show:

- All ports that have a link change from `Init` to `Active`
- Each port is assigned a LID
- The switches display a `Forwarding table`, as in the example below:

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

### Terminal 3: Run Management Tools

```bash
# Set the environment variable first
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so

# Full topology discovery
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

## 9.6 Basic Tests

### Querying Basic IB Network Information

```bash
# View all IB switches
expert@net21:~$ ibswitches
ibwarn: [6937] sim_connect: attached as client 1 at node "Switch1"
Switch  : 0x0000000000200001 ports 8 "Switch2" base port 0 lid 3 lmc 0
Switch  : 0x0000000000200000 ports 8 "Switch1" base port 0 lid 1 lmc 0

# Query the LID/GUID of all nodes
expert@net21:~$ ibnodes
ibwarn: [6944] sim_connect: attached as client 1 at node "Switch1"
Ca      : 0x0000000000100003 ports 2 "Hca2"
Ca      : 0x0000000000100000 ports 2 "Hca1"
ibwarn: [6950] sim_connect: attached as client 1 at node "Switch1"
Switch  : 0x0000000000200001 ports 8 "Switch2" base port 0 lid 3 lmc 0
Switch  : 0x0000000000200000 ports 8 "Switch1" base port 0 lid 1 lmc 0
```

### Direct SMP Queries

Send SMP MADs directly to a node, without going through the SA:

```bash
# Query port info (LID 2, port 1)
smpquery portinfo 2 1

# Query node info
smpquery nodeinfo 2

# Query switch info (LID 1, port 0)
smpquery switchinfo 1 0

# Print the linear forwarding table of all switches
dump_lfts
```

### SA Queries

The SA (Subnet Administration) is the interface through which upper-layer applications (MPI, RDMA) query paths:

```bash
# Query all node records
saquery -N

# Query all port records
saquery -P

# Query the path (the PathRecord from LID 2 to LID 4)
# A PathRecord contains the SL, MTU, rate, etc., and must be queried before RDMA connection setup
saquery -p --slid 2 --dlid 4

# Query SM info
saquery -s
```

### Path Tracing

```bash
# Trace the hop-by-hop path from LID 2 to LID 4
ibtracert 2 4

# Verify routing inside ibsim
sim> Route 2 4
```

### Topology Change Testing

Operate in the ibsim console:

```bash
# Disconnect an inter-switch link, observe the SM re-scan the network and converge
sim> Unlink "Switch1"[4]

# Restore the link, observe convergence again
sim> ReLink "Switch1"[4]

# Disconnect an entire node
sim> Unlink "Hca1"

# Restore
sim> ReLink "Hca1"
```

---

## 9.7 Quick Reference for ibsim Console Commands

| Command                              | Description                          |
| ------------------------------------ | ------------------------------------ |
| `Dump ["nodeid"]`                    | Print node/whole-network status, including LID and LFT |
| `Route <from-lid> <to-lid>`          | Verify the routing path between two LIDs |
| `Unlink "nodeid"[port]`              | Disconnect a port (simulate a link failure) |
| `Unlink "nodeid"`                    | Disconnect all links of a node       |
| `ReLink "nodeid"[port]`              | Restore a port connection            |
| `ReLink "nodeid"`                    | Restore all links of a node          |
| `Clear "nodeid"[port]`               | Disconnect and reset port state      |
| `Error "nodeid"[port] <rate> [attr]` | Inject packet-loss errors            |
| `Attached`                           | List currently connected clients     |
| `Verbose [level]`                    | Set the verbose level (0=silent, 1=debug) |
| `Start`                              | Start the network (manual start when the -s parameter is not given) |
| `Quit`                               | Exit ibsim                           |

---

## 9.8 Common Issues

### The SM Gets Stuck in the DISCOVERING State

**Cause**: the opensm version is too new (5.26.x) and incompatible with ibsim 0.12; the SM gets stuck in the SMInfo election phase.

**Solution**: downgrade to opensm 5.21.12.

### The ibsim/OpenSM Console Does Not Support Backspace

**Cause**: the ibsim console does not use the readline library, so the backspace key outputs `^H` instead of actually deleting characters.

**Solution**: start it with `rlwrap ibsim ...`.

### How to Exit ibsim/OpenSM

For ibsim, just use `quit` directly in the console to exit;

OpenSM needs to be force-quit in a new terminal with `sudo pkill -9 opensm`.

In production, OpenSM should run as a systemd service, managed in conjunction with `systemctl`, or run directly on the IB switch. For specifics, refer to the relevant support documentation.
