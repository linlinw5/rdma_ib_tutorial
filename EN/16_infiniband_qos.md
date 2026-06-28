# Chapter 16: InfiniBand Fabric QoS

## 16.1 QoS in IB

For network engineers, QoS is no stranger. So rather than spending much space introducing the definition of QoS, let's directly talk about the problem QoS sets out to solve: **when different traffic flows contend for the same link simultaneously, who goes first, who yields, and who can be guaranteed how much bandwidth?**

The QoS problem becomes very concrete in an AI cluster, where a single uplink may simultaneously carry:

- Control-plane management traffic (e.g., the SM's scanning): very small in volume, but must never be starved;
- Storage I/O: latency-sensitive;
- Training collective communication (AllReduce / All-to-All): enormous bandwidth, an out-and-out "elephant flow."

If they are crammed together without distinction, that AllReduce torrent will instantly fill up the link buffers, and the management traffic and storage I/O can only wait. This scenario is familiar to any engineer who has done Ethernet QoS: **we need to classify traffic into levels, then schedule by level.**

So the purpose of this chapter is to help everyone see clearly three things:

- How does IB solve the problem of queues and scheduling?
- How does IB make QoS work together with its credit flow-control mechanism?
- How does IB view the trust problem of QoS marking?

---

## 16.2 Two Core Concepts: SL and VL

IB's QoS is built on the division of labor between two concepts.

### SL: The "Level Tag" Stuck on the Packet

**SL (Service Level) is a 4-bit field located in the link-layer header LRH, set by the sender and kept unchanged end to end.** It can express 16 levels (SL0–SL15).

The SL itself is not any specific buffer or queue; it is just an abstract level identifier, equivalent to 802.1p PCP or DSCP in Ethernet. The sender says "this flow of mine is SL5," and this tag follows the packet all the way, with no one changing it.

So how does the sender know which SL a given flow should use?

Usually there are two ways:

- Control-plane automatic assignment. When an application queries the PathRecord via SA (Subnet Administration), the SA returns an appropriate SL for this path according to the QoS policy configured by the SM. This is the common practice in InfiniBand networks, where the traffic levels of the entire fabric are uniformly planned and pushed down by the SM.
- Application explicit specification. Some applications or communication frameworks can also directly specify the SL when creating a communication path, for example by filling in the sl field when creating an Address Handle, or by upper-layer software such as MPI or NCCL choosing a certain SL itself. In this case, the application needs to ensure that the SL it chooses matches the QoS configuration in the network, otherwise it may not get the expected quality of service.

Therefore, **the SL is essentially just a priority tag on the packet; it can either be automatically assigned by the control plane or be actively specified by the application. Once written into the LRH, it is usually no longer modified during subsequent forwarding, but follows the data packet all the way to the destination.**

### VL: The Real "Physical Queue"

**VL (Virtual Lane) is the real physical resource: each VL has its own independent set of buffers and its own independent flow-control credit on the link.** Multiple VLs multiplex the same physical link, but their buffers are independent of one another. This means that when one VL is filled up, it won't block the traffic of another VL (avoiding head-of-line blocking, HoL blocking).

> IB's credit mechanism is similar in idea to SAN networks' Buffer-to-Buffer Credit, but finer in granularity: SAN's BB credit does flow control on a per-link basis, with the entire port sharing one credit pool; whereas IB's credit is refined down to each VL, with each virtual lane independently holding its own credit quota.

IB specifies:

- **VL0 ~ VL14**: up to 15 data VLs, for ordinary traffic;
- **VL15**: reserved exclusively for management traffic (SMP, Subnet Management Packets), and **not subject to flow control**. Management traffic must be able to squeeze through under any congestion, so it travels on a special lane that is never blocked by credit.

Not all devices support a full 15 data VLs. How many a port supports is declared by the **VLCap** field in PortInfo; common values are an enumeration:

| VLCap | Supported VL |
| ----- | ------------ |
| 1     | Only VL0     |
| 2     | VL0–VL1      |
| 3     | VL0–VL3      |
| 4     | VL0–VL7      |
| 5     | VL0–VL14     |

> If a device only supports a single data lane VL0 (VLCap=1), then all SLs can only map to VL0, and all of QoS's classification and scheduling degenerate into "only one queue," rendering it useless.
>
> VLCap is only the **hardware capability upper limit**. How many VLs are actually enabled right now is another field, `OperVLs`, determined by the SM according to parameters such as `max_op_vls`; the two need not be equal.

### Why Split into Two Concepts?

At first glance it seems redundant: why not let the sender directly specify the VL instead of going through an SL?

The reason is that **the SL is unchanged end to end, while the VL is allowed to change hop by hop**. A packet may cross multiple links from source to destination, and each link may support a different number of VLs and have a different usage plan. If "level" and "which physical lane to take" were tied together, cross-device coordination would become a nightmare.

IB's approach is to decouple: **the SL is the unchanging intent of "what level am I," while the VL is the variable implementation of "on this link on this hop, which buffer do I specifically occupy."** A mapping table hooks the two together at each hop, which is the SL2VL to be introduced next.

---

## 16.3 The SL to VL Mapping (SL2VL)

Since the SL is unchanged end to end in the packet while the VL is decided hop by hop, every device, when forwarding, must answer one question: **this packet is marked SL=X, which VL of the egress link should I put it into?**

What answers this question is the **SL2VL Mapping Table (SL to VL Mapping Table)**.

Its logical structure is very simple: a 16-entry lookup table mapping each SL to a VL:

```
SL0  → VLa
SL1  → VLb
...
SL15 → VLx
```

There is a detail worth emphasizing: **the SL2VL table is organized by "ingress port + egress port,"** that is, each `[IN][OUT]SL2VL` is a 16-entry SL→VL table. For example:

```bash
# Query a switch's SL2VL table (taking OUT port 1 as an example)
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
```

> The meaning of this table is: when a data packet is about to be sent out of out port 1, the switch looks up the corresponding SL2VL entry according to the port it entered the switch through (in port) and the SL value in the packet header, thereby deciding which VL the packet should enter.
>
> In the example above, all the mapping rules are the same: even SLs (SL0, SL2, SL4...) all map to VL0, and odd SLs (SL1, SL3, SL5...) all map to VL1. So all traffic with an odd SL ultimately travels on VL1, and traffic with an even SL all travels on VL0.

Why subdivide down to port pairs? Because different links of the same switch may support different numbers of VLs, and the reasonable SL→VL mapping for traffic entering from a certain port and exiting from a certain port may not be the same. Making the mapping **path-dependent** allows for more flexible QoS and congestion-isolation strategies based on the traffic's source and destination, rather than the whole switch having just one global SL2VL table.

The end HCA side is relatively simple: it only sends packets outward and doesn't forward between ports, so it only needs one mapping. As shown in the display, its IN and OUT ports are constantly 0, i.e., `[0][0]SL2VL`.

```bash
# Query a HcaA's SL2VL table
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
```

### The Complete Journey of an IB Data Packet

From the QoS perspective, **the complete journey of a packet in the network** is like this:

1. The sender marks the SL and sends it out;
2. Each time it passes through a switch, the switch looks up the LFT by DLID to determine the egress port;
3. The switch, according to the SL in the packet, looks up `[IN][OUT]SL2VL` to decide which VL this packet enters on the next link;
4. Steps 2 and 3 repeat continuously until the destination.

```bash
#           ┌──────── Sw1 ─────────┐
# port:     1      2       3       4
#           │      │       │       │
#          HcaA   HcaB    HcaC    HcaD
# LID:      2      5       8       9
```

Take a real example, as shown above: suppose HcaA (LID 2) sends to HcaC (LID 8), marked SL=10. When this packet arrives at SW1, the switch does two independent table lookups:

- Decide which port to exit from, DLID=8: look up the LFT──► egress port = 3
- Decide which VL to enter, [IN=1][OUT=3]SL=10: look up SL2VL──► some VL

### Using VL to Avoid Deadlock

SL2VL and VL also have an important use beyond the scope of "QoS."

```bash
#        SwA ──────── SwB
#         │            │
#         │            │
#        SwD ──────── SwC
```

Remember that **credit loop (circular wait for buffer credits)** introduced in Chapter 12? At the time, we broke the loop by relying on updn routing's "no downlink-to-uplink turn." There's actually another solution: **at the turn that would form a loop, forcibly switch the packet to another VL.** Because the buffers of different VLs are independent, once the circular dependency is dispersed across different VLs, it can never close, and the loop is broken.

OpenSM's `torus-2QoS` routing engine, specially designed for torus topologies, uses precisely the approach of "two VLs per QoS level" to achieve both **deadlock freedom** and **QoS classification** at the same time.

So, **VL in IB is a resource serving dual duties: it serves both QoS classification and deadlock avoidance.** This is very different from purely QoS-oriented Ethernet queues.

> Note: in reality, the mainstream topology of IB clusters is fat-tree (various spine-leaf variants), and torus is a minority. So torus-2QoS is a very niche routing algorithm, and we won't expand on it here.

---

## 16.4 Scheduling: VL Arbitration (VLArb)

SL2VL solves "which VL a packet enters," but there's one last question: **when several VLs on the same egress port all have packets backed up, all waiting to be sent out, which VL does the port serve first?**

This is a scheduling problem, called in IB: **VL Arbitration**, controlled by the **VL Arbitration Table**. It provides both "strict priority" and "weighted fair" semantics.

> Friends familiar with Ethernet should find it no stranger, roughly similar to: Priority Queuing (PQ) and Class-Based Weighted Fair Queuing (CBWFQ).

The arbitration table is split into two sub-tables:

- **High Priority Table**
- **Low Priority Table**

Each sub-table is a string of `(VL, weight)` entries. The weight is in **units of 64 bytes**, indicating how many units of data this VL can send consecutively in one round of scheduling; this is essentially a **Weighted Round Robin**.

The relationship between the two tables is **strict priority**: the port always serves the VLs in the high-priority table first; only after the high-priority table's "quota is used up" does it become the low-priority table's turn. And this "quota" is bounded by a key field:

- **VLHighLimit (high-priority limit)**: stipulates how much high-priority traffic can be sent consecutively at most before being forced to yield to low priority.

**The existence of `VLHighLimit` is to prevent high-priority traffic from completely starving low priority**; no matter how high the priority, it cannot monopolize the link indefinitely.

> `VLHighLimit` can be queried via `smpquery PortInfo <port_ID>`.

---

## 16.5 Working Mechanism

Unlike Ethernet, the vast majority of IB configuration is pushed down by OpenSM, and QoS is no exception: all QoS configuration is computed by the SM at subnet initialization and pushed down to each port via SMP. The working mechanism is roughly as follows:

```
QoS policy (defined by admin)
      │  SM computes
      ▼
Each port's SL2VL table + VLArb table (pushed down via SMP)
      │
      ▼ for the forwarding of each packet:
  The sender tags by the SL given by the PathRecord
      → the switch looks up SL2VL to decide which VL to enter
      → the egress port, per the VLArb table, decides which VL sends first
```

---

## 16.6 Experimental Testing

This chapter continues to use that single-switch topology from before:

```bash
#           ┌──────── Sw1 ─────────┐
# port:     1      2       3       4
#           │      │       │       │
#          HcaA   HcaB    HcaC    HcaD
# LID:      2      5       8       9

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

### 16.6.1 Reading the Default Configuration

Start ibsim and OpenSM in two separate terminals:

```bash
# terminal 1:
rlwrap ibsim -s ./ib4.net

# terminal 2:
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local'
```

The commonly used commands for querying QoS are as follows:

```bash
# Query port info
smpquery PortInfo (PI) <addr> [<portnum>]

# Query the SL2VL table
smpquery SL2VLTable (SL2VL) <addr> [<portnum>]

# Query the VL arbitration table
smpquery VLArbitration (VLArb) <addr> [<portnum>]
```

Open a third terminal and view the default QoS configuration:

```bash
# Set the environment variable
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so


# Query HcaA port 1 info (LID=2)
expert@net21:~$ smpquery pi 2 1
ibwarn: [18181] sim_connect: attached as client 1 at node "Sw1"
Lid:.............................2
SMLid:...........................1
...
VLCap:...........................VL0-7                  # the hardware can enable 8 data VLs (VL0–VL7)
VLHighLimit:.....................0                      # high-priority quota limit = 0, meaning: currently all traffic goes through low-priority arbitration
VLArbHighCap:....................8                      # the high-priority VL arbitration table can hold at most 8 (VL,weight) entries
VLArbLowCap:.....................8                      # the low-priority VL arbitration table can hold at most 8 (VL,weight) entries
MtuCap:..........................2048                   # the maximum MTU the port supports (affects each VL's buffer size, indirectly related to QoS)
VLStallCount:....................7                      # after how many consecutive packets are dropped due to head-of-queue timeout, this VL is judged as "stalled"
HoqLife:.........................0                      # Head of Queue Lifetime, how long a packet can wait at most at the head of a VL queue
OperVLs:.........................VL0-1                  # currently 2 VLs are enabled, only VL0 and VL1 are lit up
...



# Query Sw1 port 1 info (switch LID=1)
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



# Query HcaA's SL2VL table (Hca LID=2, its IN and OUT ports are constantly 0, i.e., `[0][0]SL2VL`)
expert@net21:~$ smpquery sl2vl 2
ibwarn: [18208] sim_connect: attached as client 1 at node "Sw1"
# SL2VL table: Lid 2
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1| 0| 1|
# Explanation: because OperVLs=2, SLs can only land on these two, so the default SL2VL is written as SL mod 2 = 0,1,0,1...



# Query Sw1's SL2VL table (the switch displays OUT port 0 by default, i.e., the SMA Port)
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
# Explanation: switch management traffic (SMP, Subnet Management Packets) travels on VL15, so this table is really just a "default decoration" and has no actual effect.



# Query Sw1's SL2VL table (specifying OUT port 1)
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
# Explanation: because OperVLs=2, SLs can only land on these two, so the default SL2VL is written as SL mod 2 = 0,1,0,1...



# Query HcaA port 1's arbitration table
expert@net21:~$ smpquery vlarb 2 1
ibwarn: [18738] sim_connect: attached as client 1 at node "Sw1"
# VLArbitration tables: Lid 2 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x0 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# Interpretation:
# Low-priority arbitration table:
# ┌──────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
# │ Col  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ VL   │  0  │  1  │  0  │  1  │  0  │  1  │  0  │  1  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ WEI  │  0  │  4  │  4  │  4  │  4  │  4  │  4  │  4  │
# └──────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
# OperVLs=2, only two VLs are available, so we just take the two and fill the 8 slots in alternation
# VL0 cumulative weight = 0+4+4+4 = 12, VL1 = 4+4+4+4 = 16. That is, VL0 sends 12×64 bytes per round, VL1 sends 16×64 bytes, roughly 3:4, basically balanced with VL1 slightly more.
# The weight here is measured in a granularity of 64 bytes;
# So, the low-priority table = VL0 and VL1 doing weighted round robin on roughly equal footing
#
# High-priority arbitration table:
# ┌──────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
# │ Col  │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ VL   │  0  │  1  │  0  │  1  │  0  │  1  │  0  │  1  │
# ├──────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
# │ WEI  │  4  │  0  │  0  │  0  │  0  │  0  │  0  │  0  │
# └──────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
# Only the first slot (VL0, 4) has weight, the rest are all 0, looking like "VL0 has been set as high priority."
# But because VLHighLimit, this "quota," is currently set to 0, the high-priority table can't send a single byte, and the entire high-priority table is inactive.




# Query Sw1 port 1's arbitration table
expert@net21:~$ smpquery vlarb 1 1
ibwarn: [18740] sim_connect: attached as client 1 at node "Sw1"
# VLArbitration tables: Lid 1 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x0 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |0x4 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |0x0 |0x1 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# Interpretation same as above, no need to repeat
```

### 16.6.2 OpenSM Configuration Options

Starting from this chapter, OpenSM's configuration options are getting more and more numerous, and it's no longer convenient to continue using the simple command-line parameter form like in the previous chapters. Let's switch to a production-style configuration method: write OpenSM's configuration into an `opensm.conf` file, then load it when starting OpenSM.

> The default storage path for `opensm.conf` is `/etc/opensm/opensm.conf`, but you can also use the `-F` parameter to specify loading a particular config file when starting OpenSM.

```bash
sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so rlwrap opensm -q local -F ./opensm.conf'
```

If you want to understand all the configuration options of `opensm.conf`, you can use `opensm --create-config ./opensm-full.conf` to generate a config file template.

[opensm-full-template](../conf/opensm_full.conf).

The main configuration excerpt and explanation of the QoS part is as follows:

```conf

# qos_* (global default, the fallback default value for all ports)
#     └── qos_ca_*       overrides the HCA ports of Channel Adapters
#     └── qos_sw0_*      overrides switch port 0 (the management port)
#     └── qos_swe_*      overrides all switch external ports
#         └── qos_sw2sw_*    further overrides the switch-to-switch ports among them
#     └── qos_rtr_*      overrides Router ports

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

---

### 16.6.3 The Requirement Scenario

Next, let's assume a requirement scenario and then see IB QoS's solution.

Imagine a shared IB subnet of an AI training cluster, where a certain link simultaneously carries three kinds of RDMA traffic of differing nature:

- **Storage and Checkpoint**: during training, the framework periodically writes the model weights to distributed file systems such as Lustre and GPFS, using the NFS over RDMA or NVMe-oF protocol. This kind of traffic's bandwidth demand is not the highest, but it has a strict time window; if a Checkpoint write is blocked for a long time and exceeds the framework's timeout threshold, the entire training job fails outright.
- **AllReduce gradient synchronization**: gradient aggregation between GPUs is the main bandwidth consumer of the entire link, and also a typical elephant flow—highly bursty, and once it starts it continuously occupies a large amount of buffer.
- **AllGather and background traffic**: parameter broadcast, model export, monitoring collection, etc. Its bandwidth demand is lower than AllReduce, but it is likewise bursty, with the lowest priority; best-effort is fine, but it must not be completely starved either.

It should be specially noted that AI training frameworks' **coordination and signaling** (task scheduling, barrier negotiation, heartbeat detection) usually travel over the TCP/IP out-of-band network and do not pass through the IB RDMA data path, so they are not within the scope of our QoS design.

The network team puts forward three specific requirements:

1. **Isolation**: the three kinds of traffic must use their own independent buffers, to prevent the AllReduce elephant flow from head-of-line blocking the Checkpoint traffic;
2. **Priority**: storage traffic takes precedence over training traffic, able to "cut the line" in the queue, guaranteeing the latency of Checkpoint writes;
3. **Bandwidth allocation + starvation prevention**: normally bandwidth is mainly given to AllReduce; when storage traffic appears it can grab bandwidth; but storage traffic must not completely starve AllReduce and AllGather, and must ensure they periodically get serviced.

### 16.6.4 Translating the Requirements into IB QoS Configuration

These three requirements happen to correspond one-to-one to the three mechanisms we discussed earlier:

| Requirement              | Which mechanism to use                   | How to configure                                              |
| ------------------------ | ---------------------------------------- | ------------------------------------------------------------- |
| Isolation                | **SL2VL**                                | Map the SLs of the three traffic types to different VLs, naturally obtaining independent buffers |
| Priority                 | **VLArb high-priority table**            | The storage traffic's VL goes into the high table, training and background in the low table |
| Bandwidth + anti-starvation | **VLArb low-priority weight + VLHighLimit** | AllReduce takes a large weight in the low table, and VLHighLimit limits the high table's max consumption per round |

**The SL-to-VL division:**

The 16 SLs are divided into three segments:

| SL range | Traffic type                | Maps to |
| -------- | --------------------------- | ------- |
| SL 0–5   | Storage/Checkpoint          | **VL0** |
| SL 6–10  | AllReduce gradient sync     | **VL1** |
| SL 11–15 | AllGather / background      | **VL2** |

**VLArb table design:**

VL0 (storage) goes alone into the high-priority table, while VL1 and VL2 rotate by weight in the low-priority table:

- **High-priority table** (`qos_vlarb_high`): only VL0, weight 4. As long as the high table has data, the scheduler serves VL0 first, ensuring Checkpoint writes are not preempted by training traffic.
- **Low-priority table** (`qos_vlarb_low`): VL1 weight 28, VL2 weight 4. AllReduce gets 87.5% of the low table's bandwidth share, and background traffic gets 12.5%.
- **VLHighLimit = 4**: the high table consumes at most 4 credits consecutively per round, and after reaching the cumulative limit, the scheduler is forced to switch to serve the low table once, after which the high table resets its count. This mechanism ensures that even if storage traffic is continuous, AllReduce will inevitably get sending opportunities periodically and won't be starved.

The final opensm.conf configuration is as follows:

```bash
# ~/opensm.conf
# opensm by default tries to enable `ar_updn`; this case specifies the routing engine as minhop, unrelated to qos;
routing_engine minhop

qos TRUE
# Enable the QoS feature, so opensm will push down the SL2VL table and VLArb table

max_op_vls          4
# The enumeration code of the OperVLs field; 4 corresponds to enabling VL0–VL7 (8 VLs),
# enough to cover VL0, VL1, VL2 used in our design

qos_max_vls         8
# The maximum number of VLs the subnet allows to be negotiated, filled in directly as a number (not an enum code),
# semantically consistent with max_op_vls 4

qos_sl2vl           0,0,0,0,0,0,1,1,1,1,1,2,2,2,2,2
# The mapping of the 16 SLs to VLs, corresponding to SL0–SL15 in order:

qos_vlarb_high      0:4
# High-priority arbitration table: only VL0, weight 4
# The format is VL:weight; the scheduler prefers to pick packets from the high table to send
# The valid range of weight is 0–255 (uint8_t)

qos_vlarb_low       1:28,2:4
# Low-priority arbitration table: VL1 weight 28, VL2 weight 4
# VL1 takes 87.5% of the low table's bandwidth, VL2 takes 12.5%

qos_high_limit      4
# The global default value of VLHighLimit: the high-priority table consumes at most 4 credits consecutively per round,
# and after reaching the limit the scheduler is forced to serve the low table once, preventing VL0 from starving VL1/VL2
```

### 16.6.5 Verification One: Interface Information

Start OpenSM:

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

Next, open a new terminal and read PortInfo's `OperVLs` and `VLHighLimit`:

```bash
# Load the environment variable
expert@net21:~$ export LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so

# Query HcaA port 1's port info
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
VLHighLimit:.....................0                         # VLHighLimit didn't take effect; it should be 4. This is a known ibsim issue
VLArbHighCap:....................8
VLArbLowCap:....................8
InitReply:.......................0x00
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................0
OperVLs:.........................VL0-7                     # 8 VLs are already enabled
...

# Query Sw1 port 1's port info
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
VLHighLimit:.....................0                         # VLHighLimit didn't take effect; it should be 4. This is a known ibsim issue
VLArbHighCap:....................8
VLArbLowCap:....................8
InitReply:.......................0x00
MtuCap:..........................2048
VLStallCount:....................7
HoqLife:.........................16
OperVLs:.........................VL0-7                     # 8 VLs are already enabled
...
```

> ibsim currently does not support handling VLHighLimit. This can be seen from the source code: in sim_net.c lines 1269–1307, update_portinfo() is the function called before each GET/SET response, responsible for writing the Port struct's fields back to p->portinfo[]. Currently ibsim does not process the VLHighLimit field; this is a simulator limitation.
>
> https://github.com/linux-rdma/ibsim/blob/master/ibsim/sim_net.c#L1269
>
> So here we can only understand it in spirit and can't see the experimental effect.

### 16.6.6 Verification Two: SL2VL Has Split the Two Traffic Types into Different VLs

After applying the policy, look at the SL2VL table:

```bash
# HcaA's SL2VL table:
expert@net21:~$ smpquery SL2VLTable 2
ibwarn: [20792] sim_connect: attached as client 2 at node "Sw1"
# SL2VL table: Lid 2
#                 SL: | 0| 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|
ports: in  0, out  0: | 0| 0| 0| 0| 0| 0| 1| 1| 1| 1| 1| 2| 2| 2| 2| 2|


# Sw1's SL2VL table:
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

The mapping has changed from the baseline alternating `0,1,0,1...` to our designed **SL0–5 → VL0, SL6–10 → VL1, SL11–15 → VL2**, consistent with the `qos_sl2vl` configuration.

### 16.6.7 Verification Three: The VLArb Table

After applying the policy, look at the arbitration table of each interface:

```bash
# HcaA port 1's arbitration table
expert@net21:~$ smpquery VLArbitration 2 1
ibwarn: [20797] sim_connect: attached as client 2 at node "Sw1"
# VLArbitration tables: Lid 2 port 1 LowCap 8 HighCap 8
# Low priority VL Arbitration Table:
VL    : |0x1 |0x2 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x1C|0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
# High priority VL Arbitration Table:
VL    : |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |
WEIGHT: |0x4 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |0x0 |

# Sw1 port 1's arbitration table
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

The output is in hexadecimal; after conversion, comparing each item against our configuration, all are consistent:

| Table         | Hex                 | Dec                   | Qos Conf                 |
| ------------- | ------------------- | --------------------- | ------------------------ |
| High priority | `VL0:0x4`           | VL0=**4**             | `qos_vlarb_high 0:4`     |
| Low priority  | `VL1:0x1C, VL2:0x4` | VL1=**28**, VL2=**4** | `qos_vlarb_low 1:28,2:4` |

---

## 16.7 Testing in a Real Environment

Since ibsim has no data plane, here we can only verify the configuration on OpenSM's control plane.

In reality, if you have IB hardware and want to truly observe the effect of QoS, you can use a tool like `ib_send_bw` to push traffic on different SLs simultaneously (the `--sl` parameter specifies the SL, defaulting to 0), then use the port counter `perfquery` to observe the actual bytes carried by each VL. This part is left to be verified when hardware is available.

---

## 16.8 The Assignment and Management of SL

By now, IB QoS's queue and scheduling mechanisms have been clearly introduced. But friends familiar with Ethernet should still have one question: how is IB's QoS marking done?

I'd like to emphasize once more: InfiniBand's QoS mechanism has a fundamental premise in its architecture: the SL field in the packet header is filled in by the **sender** when setting up the QP, and the switch will not modify it during forwarding—it only makes scheduling decisions based on it. This means the validity of the entire QoS system, from the very start, is built on trust in the compliant behavior of the ULP: an application can completely ignore any policy and fill in an arbitrary SL value when setting up a QP.

This forms an essential difference from Ethernet's QoS mechanism. An Ethernet switch can forcibly rewrite a packet's DSCP or 802.1p marking at the ingress port, with the QoS marking enforcement point on the network device, hard for the application to bypass. But in an IB network, "marking" happens during the PathRecord query stage before connection setup; the SM "suggests" that the application use a certain service level by returning a PathRecord carrying a specific SL, but whether this suggestion is followed depends entirely on the ULP's behavior. The comparison of the two is shown in the table below:

|                              | Ethernet                          | InfiniBand                          |
| ---------------------------- | --------------------------------- | ----------------------------------- |
| Marking happens at           | Port ingress direction (can force rewrite) | When the PathRecord is returned (before connection setup) |
| Can the data plane change the marking | Yes (switch rewrites DSCP) | No (IB switch doesn't change SL)    |
| Can the application bypass it | Hard (switch enforces)           | Yes (ignore the SL in the PathRecord) |
| Normal operation depends on  | Switch policy                     | ULP compliant behavior              |

As mentioned at the start of this chapter when introducing the SL, in actual deployment, the assignment of SL is usually achieved through two paths:

- The first is **directly hard-coded or specified via environment variables** by the application or framework layer. Take NCCL, widely used in AI training scenarios, as an example: during the initialization stage it needs to quickly establish a large number of RC QPs, and to avoid the latency brought by querying the SA one by one, NCCL directly fills in the SL value when setting up the QP, defaulting to 0, and opens an override entry to operators via the environment variable `NCCL_IB_SL`. This approach is simple and direct, but the SL assignment strategy must be guaranteed consistent at the cluster level through operational convention.

- The second relies on **the SM's advanced QoS policy**. OpenSM supports defining a set of matching rules in `qos-policy.conf`: grouping ports by GUID, node type, or the Partition they belong to, then mapping different types of PathRecord queries to predefined SLs by traffic characteristics (source/destination port groups, PKey, Service ID). ULPs going down this path (such as MPI, SRP, iSER) don't need to be aware of the existence of SL when initiating a path query; the SM has already transparently completed the traffic classification and marking when returning the PathRecord.

The two approaches often coexist in large AI clusters: NCCL training traffic goes through a specific SL by environment-variable convention, while storage and management traffic are automatically classified through SM policy, each managing its own independent portion of traffic.

Limited by the length of this chapter, we'll set aside expanding on "advanced QoS policy" for now.

---

## 16.9 Summary

**SL and VL are two sides of the same coin in IB QoS**: SL is the "level tag" stuck on the packet, unchanged end to end; VL is the "physical queue" on the link with independent buffers and independent credit (VL0–14 for data + VL15 for management). SL expresses intent, VL implements resources, and the two are deliberately decoupled to adapt to links that change hop by hop.

**Two tables glue them together**: the SL2VL mapping table answers "which VL this SL should enter" (subdivided by ingress/egress port on switches, with SL unchanged but VL changeable hop by hop); the VLArb arbitration table is responsible for scheduling—strict priority + weighted round robin + starvation prevention. This mechanism, combined with IB's credit flow control, lets IB perform freely in AI training and high-performance computing scenarios.

**QoS marking**: IB's QoS has an implicit premise: the SL is filled in by the sender (ULP) itself when setting up the QP, and the switch only reads it, never changes it, all the way along. So the validity of the entire QoS rests on ULP compliance: the SL returned by the SM via the PathRecord is only a "suggestion," which the application can completely ignore. In practice, the assignment of SL goes two ways: either the application layer specifies it directly, or it's left to the SM's `qos-policy.conf` to automatically classify by traffic characteristics and fill it into the PathRecord.

**IB has taken "queuing and scheduling" to the extreme of centralization and determinism, yet has handed back the first step of "who should use which SL" to trust on the end side.**

At this point, we can both "partition tenants and isolate traffic" and "classify traffic into levels and schedule by level" on an IB fabric.
