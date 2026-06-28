## Chapter 14: InfiniBand Fabric Adaptive Routing

### 14.1 The Boundaries of Static Routing

In the previous chapter, we observed through experiments the behavioral differences between the updn and ftree routing algorithms under the same spine-leaf topology. Although the two differ in their LFT-generation logic, they share one fundamental point in common: **the routing table is computed and written once by the SM at subnet initialization, and is fixed thereafter.**

This means that when the SM computes routes, it faces a static topology graph, not a dynamically changing traffic matrix. All it can do is make a static pre-allocation among all possible paths.

This approach performs well when traffic is uniformly distributed. But in actual AI training scenarios, traffic is rarely uniform. Take AllReduce communication as an example: when multiple nodes simultaneously send gradient data to the same target, some Spines instantly become hot spots, while at the same time other Spines may be almost idle. Static routing is helpless against this; even if a link is already congested, the switch still continues to forward according to the port hard-wired in the LFT, and won't proactively route around it.

So the essence of the problem is: **the balancing granularity of static routing is "at subnet initialization," while traffic imbalance occurs "at each packet forwarding."** The enormous gap between these two time scales is precisely the fundamental limitation of static routing.

---

### 14.2 The Idea of Adaptive Routing

The idea for solving this problem is not complicated: since the SM cannot foresee the runtime traffic distribution at initialization, then delegate part of the decision-making power to the switch, letting it choose the exit based on the actual current load state of each port when forwarding each packet. This is the core idea of Adaptive Routing (AR).

**AR does not make the SM withdraw from routing decisions; rather, it changes the division of labor between the SM and the switch:**

```
Without AR:
  SM  →  compute which port each LID takes  →  write the LFT  →  switch forwards by the table

With AR:
  SM  →  compute which ports are equivalent paths  →  write the AR Group Table
  Switch  →  perceive each port's load at runtime  →  dynamically choose a port from the Group
```

The SM no longer tells the switch "to LID X, go out port 3," but tells it "to LID X you can go out port 1, 2, 3, or 4; they are all equivalent, so decide for yourself."

The switch, on each forward, reads the real-time credit or queue depth of each candidate port and chooses the one that is currently most idle.

Therefore, **AR shortens the time granularity of routing decisions from "at subnet initialization" to "at each packet forwarding."**

---

### 14.3 The Structure of the AR Group Table

The implementation vehicle of AR is the AR Group Table, which exists in the switch alongside the standard LFT.

The structure of the standard LFT is:

```
LID → single exit port
```

The structure of the AR Group Table is:

```
LID → AR Group ID → { port_1, port_2, port_3, ... }
```

Through the ARM (Adaptive Routing Manager) module, the SM scans all AR-capable switches at subnet initialization, identifies equivalent uplinks, groups them into the same AR group, and then writes the group configuration into the switch. After that, the SM no longer participates in per-packet forwarding decisions; this part is completed entirely autonomously by the switch hardware.

Take the spine-leaf topology from the previous chapter as an example: each Leaf switch has 4 uplinks connecting to 4 Spines respectively, so the AR group configuration is roughly as follows:

```
Leaf1 AR Group 0: { port1, port2, port3, port4 }   ← 4 equivalent uplinks

LFT[lid_X] → Group 0    (all cross-Leaf target LIDs point to this group)
```

When forwarding, the switch no longer looks up the LFT to get a fixed port, but looks up Group 0 and then chooses, in real time, the one of the 4 ports with the lowest load to forward out of.

---

### 14.4 The Cost of AR: The Packet Reordering Problem

**Making AR run perfectly is not without cost.** In the analysis of the previous chapters, we know that the IB protocol is designed to guarantee strict ordering of messages within the same QP: the packets the sender sends in order are received by the receiver in the same order. The premise of this guarantee is: all packets of the same connection take the same path.

AR breaks this premise. With AR enabled, two consecutive packets of the same QP may take different Spines due to instantaneous changes in port load, experience different queuing delays, and arrive out of order at the receiver.

Therefore, to support AR, the HCA needs to have packet reordering capability, rearranging the out-of-order packets at the receiving end before handing them to the upper layer. This imposes additional requirements on the HCA's hardware design, and is one of the reasons early IB devices did not support AR. Modern NVIDIA ConnectX-series HCAs all support packet reordering, and only then could AR be widely deployed in production environments.

> AR's forwarding decision occurs when the switch processes each packet, with a theoretical granularity of per-packet. But because of the ordering requirement of IB RC QPs, in actual deployment the HCA needs to have packet reordering capability. That said, the specific implementation details of NVIDIA are proprietary technology, and whether it makes conservative decisions at a flow (QP) granularity in certain scenarios is something for which I currently have no complete public documentation.

---

### 14.5 Experimental Testing

Next, we observe through experiments the differences between ar_ftree and ftree on the control plane. Unfortunately, ibsim does not support Adaptive Routing. AR relies on the proprietary extension capabilities of NVIDIA IB switches, which ibsim has not implemented. So what we can observe is limited to the behavior of the OpenSM control plane, and we cannot see the dynamic decision process of the switch data plane.

First, start ibsim as usual:

```bash
expert@net21:~$ rlwrap ibsim -s ./ib3.net
```

Then, start OpenSM, specifying the ar_ftree routing engine:

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

Compared with running ordinary ftree in the previous chapter, the following lines in the log are worth attention:

```bash
AR Manager - Configuration cycle (number 1) completed successfully
osm_ucast_mgr_process: ar_ftree tables configured on all switches
osm_ar_calculate_pfrn: No fabric switch supports pFRN. Hence, avoid configuring pFRN
```

**The first line** indicates that the AR Manager (ARM) module was activated and completed one configuration cycle. The ARM is the component in OpenSM responsible for AR configuration; after route computation completes, it scans all switches in the fabric, identifies AR-capable devices, and writes the AR Group Table to them. "completed successfully" means this process ran through normally. But note: "completed successfully" only means the process did not error out, and does not mean the AR configuration actually took effect.

**The third line** is the real conclusion: `No fabric switch supports pFRN`. pFRN (port-based Forwarding Route Notification) is a companion feature of AR, and here it clearly states that no switch in the fabric declared this capability. In other words, the ARM scanned all the virtual switches, found that none of them had AR hardware capability, and so skipped writing the AR Group Table, configuring nothing.

At this point, if you use `ibroute` to view each Leaf switch's forwarding table, you'll find the result is completely identical to ordinary ftree. This is the expected result, because in the absence of an AR Group Table, packets are still forwarded according to the ordinary LFT, and ar_ftree degenerates into ftree.

---

### 14.6 The Boundaries of ibsim

We are gradually beginning to approach the limits of ibsim's capabilities, and you'll see many similar situations in the chapters that follow, so here I'd like to specifically discuss ibsim's limitations.

**What can be observed (control plane):**

OpenSM identifying the fat-tree topology structure, computing routes, generating the LFT, calling the ARM module, attempting to configure the AR Group Table—this entire control-plane process can run and be observed fully in ibsim. The topology identification information in the log (rank, leaf/root counts), and the ARM's startup and run records, are all real control-plane behavior.

**What cannot be observed (data plane):**

| What you want to observe                        | Can ibsim do it                        |
| ----------------------------------------------- | -------------------------------------- |
| The actual content of the AR Group Table        | ❌ virtual switches don't support AR capability |
| Which port the switch chooses from the group at runtime | ❌ ibsim has no data plane       |
| How much congestion AR actually reduces         | ❌                                     |
| Switching behavior when a port's load exceeds the threshold | ❌                          |
| The occurrence rate of out-of-order packets     | ❌                                     |

To observe these, you need real NVIDIA IB switches, working with UFM's AR monitoring interface or the `ibdiagnet --ar_groups` command, to see the AR Group Table's configured content and runtime port-selection statistics.

---

### 14.7 Summary

We have now fully walked through the evolutionary path of IB routing from static to dynamic.

updn and ftree both belong to static routing: the SM computes routes and writes the LFT at subnet initialization, after which the routing table is fixed.

ar_updn and ar_ftree introduce adaptive routing: the SM additionally computes which ports are equivalent uplinks, writing the AR Group Table via the ARM; the switch, when forwarding, selects in real time the port with the lowest load from the group. This is the fundamental means of coping with the bursty traffic imbalance in AI training scenarios.

AR's real value manifests on the data plane: when collective communications such as AllReduce and All-to-All are conducted simultaneously among thousands of GPUs, AR's perception of real-time traffic and its dynamic balancing are precisely the key to the IB network being able to fully exert its capabilities.
