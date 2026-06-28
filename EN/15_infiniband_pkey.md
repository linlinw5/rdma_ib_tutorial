# Chapter 15: InfiniBand Fabric Partitions

## 15.1 Starting from a Familiar Problem

So far, the IB subnets we've built share one common characteristic: **all nodes are reachable to one another, and anyone can send packets to anyone.** The Subnet Manager computes the LIDs, fills in the forwarding tables, and the whole fabric is a flat, fully connected layer-2 network.

But here I must point out specifically: **"mutually reachable" does not mean there's a "network-wide broadcast" mechanism in IB like there is in Ethernet.** As everyone knows, Ethernet's address discovery (ARP) relies on layer-2 broadcast: when you don't know the peer's MAC, you shout across the broadcast domain "who is this IP," and wait for a response. IB has no such broadcast-style address discovery.

Recall what was covered in Chapter 2: to initiate IB communication, the initiator needs to already hold some kind of address identifier for the peer—usually an IP address or GID, which ultimately must be converted into a LID for addressing. In actual engineering, the source of this LID is roughly threefold:

- Relying on an out-of-band TCP channel to advertise the peer's parameters (including the LID) in advance (the approach commonly used by AI training frameworks such as NCCL);
- Relying on RDMA CM (rdma_resolve_addr + rdma_resolve_route) to obtain it via an SA PathRecord query before connection setup;
- Or, in testing and HPC scenarios, statically writing it in directly.

In other words, the "full connectivity" within an IB subnet means "as long as you already hold the peer's LID or GID, the SA can definitely return a complete path record, and the forwarding tables can definitely get the packet there," not "you can broadcast to discover anyone in the network."

> Strictly speaking, IB is not entirely without "one-to-many" capability; it has multicast explicitly managed by the SM (multicast, MGID/MLID), and IPoIB precisely borrows a multicast group to simulate Ethernet's broadcast semantics, while ibacm also uses IB multicast for ARP-like address resolution. But these are all mechanisms that require SM allocation and explicit application-level joining, completely dependent on centralized management—an entirely different thing from Ethernet's implicit broadcast where "by default everyone receives it." This usage is almost never seen in mainstream AI/HPC workloads.

**Real data centers' demands on the network often don't stop at "interconnectivity"; they often also have demands around "partitioning and control."** For example, a single physical network often runs multiple mutually distrusting tenants and several unrelated workloads: Department A's training cluster shouldn't see Department B's storage traffic, and one GPU tenant's AllReduce shouldn't accidentally land on another tenant's node.

In the Ethernet world, the tool to solve this problem is one we know all too well: **VLAN**. With a single physical cable and a single physical switch, by tagging ports with different VLAN IDs, you can slice one physical network into several mutually isolated broadcast domains, and the VLAN division decides who can communicate with whom at layer 2.

InfiniBand faces the same demand and gives an answer that is **functionally highly similar but mechanistically completely different**: **Partitions**, whose vehicle is a 16-bit identifier called the **P_Key (Partition Key)**.

That said, when using VLAN as this analogy, one thing must be said up front, otherwise it's easy to be led astray: **VLAN was originally designed to isolate broadcast domains.** Ethernet has too much broadcast, so a large layer-2 domain needs to be sliced into smaller pieces to suppress broadcast storms and shrink fault domains, so isolating "who can communicate with whom" is more of an incidental result obtained after slicing the broadcast domain. IB never had broadcast in the first place, so the starting point of partitions was, from the very beginning, **not "suppressing broadcast" but "controlling who can communicate with whom."**

So purely in terms of purpose, IB's partitions are actually closer to **access control (ACL)**; if you have a storage networking background, you'll find it more like **Zoning in a SAN**: on a shared physical fabric, explicitly stipulating which ports are allowed to see and communicate with each other, with mutual invisibility by default. So **VLAN is "slicing broadcast domains," and partitions are "defining reachability relationships."** Remember this difference in starting point, and the later designs—membership bits, the default partition—will all make sense as to why they are the way they are.

---

## 15.2 P_Key: The Vehicle of Partitions

Let's first establish the most core object.

**A P_Key is a 16-bit value, and every IB port holds a P_Key Table recording "which partitions I belong to."** A port can belong to multiple partitions at the same time, just as an Ethernet trunk port can carry multiple VLANs.

These 16 bits are not a flat single number; they are split into two parts:

```
bit15        bit14 ........................ bit0
┌──────────┐ ┌──────────────────────────────────┐
│ Member   │ │     15-bit partition key value   │
│ (1 bit)  │ │          (P_Key value)           │
└──────────┘ └──────────────────────────────────┘
```

- **The low 15 bits**: these are the key value that actually distinguishes "which partition." For two ports to communicate, these 15 bits must be equal, equivalent to VLAN IDs needing to match.
- **The highest bit (bit 15)**: is the **membership bit**, which does not participate in the judgment of "whether it's the same partition," but marks "what identity I have in this partition":
  - `1` = **Full member**
  - `0` = **Limited member**

The specific role of this membership bit is introduced in the next section. For now, just remember a conversion: for the same partition key `0x0100`, a full member is stored in the P_Key table as `0x8100` (highest bit set to 1), and a limited member is stored as `0x0100`.

> When we introduced the BTH (Base Transport Header) in Chapter 10, we focused on the fields DLID/SLID, QPN, and PSN. In fact, the BTH also has a 16-bit field we didn't expand on at the time, namely the **P_Key**. **Every IB data packet carries, in the BTH, the P_Key of the partition it belongs to.** This is the physical basis for partition isolation taking effect on every packet: the receiver gets the packet and can read from the BTH which partition it came from, then compare it against its own P_Key table to decide whether to receive or drop it.

---

## 15.3 Full and Limited: An Asymmetric Isolation Rule

If IB partitioning were merely "communication is possible only when key values are equal," then it would be almost no different from VLAN. But the introduction of the membership bit gives IB an extra layer of new capability: **within the same partition, it can further distinguish two kinds of members, "those that can communicate with each other" and "those that cannot."**

The rule is just one sentence, but look closely:

> **Two ports can communicate if and only if: the low 15 bits of their P_Keys are equal, and at least one of the two is a full member.**

Expand it into a table:

| Initiator \ Peer        | Full member      | Limited member      |
| ----------------------- | ---------------- | ------------------- |
| **Full member**         | ✅ can communicate | ✅ can communicate |
| **Limited member**      | ✅ can communicate | ❌ **cannot communicate** |

In other words:

- **Full members** communicate freely with each other;
- **Full member ↔ limited member** can communicate;
- **Limited member ↔ limited member**—even within the same partition, with completely identical key values—**cannot** communicate.

> Heh, networking folks should immediately realize—what kind of "new capability" is this? Isn't it just Ethernet's `Private VLAN`? Indeed it is.

The practical use of this rule is very typical: **doing "many-to-one" star isolation.** For example, a set of shared storage: the storage node is set as a full member, and all clients as limited members. The result is: every client can access the storage (limited ↔ full, connected), but the clients can't see each other (limited ↔ limited, not connected). You don't need to create a separate partition for each pair of "clients that should be isolated"; one partition + the membership bit handles it.

---

## 15.4 The Default Partition and the Management Channel: Why "After Partitioning, It Can Still Be Managed by the SM"

By this point, you may have a question: **If I divide all ports into their respective tenant partitions, how can the Subnet Manager (SM) still communicate with them? And which partition does the SM itself belong to?**

IB has a fallback design for this.

**The Default Partition.** There is a reserved partition key `0x7FFF`, called the default partition. In the absence of any partition configuration, all ports are full members of the default partition. This is why, even though we never configured partitions earlier, the whole network was fully connected: everyone was actually staying in the same default partition.

- `0xFFFF`: a **full member** of the default partition
- `0x7FFF`: a **limited member** of the default partition

**Management traffic travels over the default partition.** Management entities such as the SM and SA communicate with all nodes precisely through the default partition. So the default partition cannot be broken, otherwise once the SM divides a node out, it can never manage it again and the subnet directly loses contact.

OpenSM has a key **protective behavior** for this: when you define tenant partitions and remove a port from the default partition's full members, OpenSM still **automatically retains the default partition's limited membership (`0x7FFF`) for it**. This way:

- Between this port and the SM (a default-partition full member): limited ↔ full, **connected**—the management channel is preserved;
- Between this port and other tenants' ports (if the other side is also only a default-partition limited member): limited ↔ limited, **not connected**—tenant isolation also holds.

The asymmetric rule of the membership bit is used to the fullest by OpenSM here: **it keeps the management plane always reachable while not breaking the data-plane isolation between tenants.**

---

## 15.5 Where the P_Key Table Lands: Ports, Switches, and "Enforcement"

The partition configuration ultimately has to become a P_Key table on each port. This step is completed by the SM at subnet initialization, and the mechanism is the same as the LFT push-down we saw in the previous chapters: **the SM computes which P_Keys each port should have and writes them in via the SMP (Subnet Management Packet) Set operation (the PKeyTable attribute).**

But "holding a P_Key table" and "actually enforcing isolation" are two different things, and this must be distinguished here:

- **HCA ports (end nodes)**: P_Key checking is **Mandatory**. When the NIC sends a packet, it fills the P_Key into the BTH; when receiving a packet, it compares the P_Key in the packet against its own table, dropping it if there's no match, and may report a "P_Key violation" trap to the SM. This is mandatory and cannot be turned off.
- **Switch external ports**: P_Key enforcement is **Optional** (PartitionEnforcementInbound / PartitionEnforcementOutbound). If the switch supports it and enables it, it does P_Key filtering on the spot during forwarding, intercepting non-compliant packets midway through the network, not letting them needlessly consume bandwidth all the way to the destination; if not supported, isolation is left entirely to the HCAs at both ends as a fallback.
  - Inbound: P_Key checking on packets entering the switch from this port. That is, traffic sent from an external HCA that has just arrived at this port is checked for P_Key before being forwarded.
  - Outbound: P_Key checking on packets leaving the switch from this port. That is, traffic the switch has already decided to forward out this port is checked for P_Key once more before actually being sent.

This division of labor of "mandatory at end nodes, optional at switches" is exactly the opposite of the Ethernet intuition that "VLAN filtering is mainly done on switch ports," and is worth noting: **the ultimate, fallback enforcer of IB partition isolation is the end NIC, not the switch.**

As for how many partitions a P_Key table can hold, that is determined by the port's PartitionCap attribute (commonly 128 for HCAs), with table entries organized in blocks of 32. These are implementation details; it's enough to know they exist.

---

## 15.6 Experimental Testing

First, here is the address of the OpenSM help documentation, where you can find the relevant format description of partitions.conf:

https://manpages.debian.org/testing/opensm/opensm.8.en.html

Second, the partition-related parameters when starting opensm:

```bash
# -P is used to specify the address of the partitions.conf config file
--Pconfig, -P <partition-config-file>
          This option defines the optional partition configuration file.
          The default name is '/etc/opensm/partitions.conf'.

# -Z is used to specify the switch's enforcement policy; it has no effect for ibsim
--part_enforce, -Z [both, in, out, off]
          This option indicates the partition enforcement type (for switches)
          Enforcement type can be outbound only (out), inbound only (in), both or
          disabled (off). Default is both.
```

We design an ibsim topology specifically for the experiments that follow:

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

### 15.6.1 Test One: Observing the Default Partition Table

Start the simulator and `dump`:

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

Record each Hca's GUID as follows:

| Node | Port GUID |
| ---- | --------- |
| HcaA | 0x100001  |
| HcaB | 0x100004  |
| HcaC | 0x100007  |
| HcaD | 0x10000a  |

Start OpenSM, for now without loading any partition configuration:

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -f -'
```

Observe ibsim; the LID list is as follows:

| Node | LID |
| ---- | --- |
| Sw1  | 1   |
| HcaA | 2   |
| HcaB | 5   |
| HcaC | 8   |
| HcaD | 9   |

Use `smpquery pkeys` to read the P_Key table each port actually obtained:

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

As you can see, for the devices simulated by ibsim, the switch has 8 pkeys capacity by default, while the HCAs have 64 pkeys capacity by default.

Currently all devices have been automatically assigned a default partition key: 0xffff, indicating that everyone is a "full member of the default partition."

This is equivalent to a partitions.conf configured as:

```conf
# p1.conf
#
# keywords for PortGUID definition:
# - 'ALL' means all end ports in this subnet.
# - 'ALL_CAS' means all Channel Adapter end ports in this subnet.
# - 'ALL_SWITCHES' means all Switch end ports in this subnet.
# - 'ALL_ROUTERS' means all Router end ports in this subnet.
# - 'SELF' means subnet manager's port.

# All HCAs are in the default partition, full members
Default=0x7fff : ALL=full ;
```

Per the explanation above, if you want to set all the Hcas as limited members of the default partition, you can write it like this:

```conf
# p2.conf

Default=0x7fff : ALL=limited, SELF=full ;
```

### 15.6.2 Test Two: Slicing One Fabric into Two Tenants

Based on the Hca port GUIDs dumped in the previous section, write a new partitions configuration:

```conf
# p3.conf

# Partition A: HcaA + HcaB
PartA=0x0010 : 0x100001=full, 0x100004=full ;

# Partition B: HcaC + HcaD
PartB=0x0020 : 0x100007=full, 0x10000a=full ;

# The default partition is given only to the SM itself, no other nodes added
Default=0x7fff : SELF=full ;
```

Start OpenSM (loading the custom partition file):

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -P p3.conf -f -'
```

Use `smpquery pkeys` to read the P_Key table each port actually obtained:

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

From the output above, it's easy to see that besides obtaining `0x7fff` (limited member of the default partition), each Hca also obtained full membership of its respective partition (`0x8010` or `0x8020`).

### 15.6.3 Test Three: The Difference Between full and limited Members

Imagine a shared-storage model: HcaA acts as the storage node (full member), HcaB / HcaC act as two mutually distrusting clients (limited members), and HcaD is not allowed to access the storage.
Rewrite `partitions.conf`:

```conf
# p4.conf

# HcaA is a full member, HcaB/HcaC are limited members, HcaD is not in this partition
# limited members can't communicate with each other, but both can communicate with HcaA
PartC=0x0030 : 0x100001=full, 0x100004=limited, 0x100007=limited ;

# Default partition: assign full to the SM itself
Default=0x7fff : SELF=full ;
```

Start OpenSM (loading the custom partition file):

```bash
expert@net21:~$ sudo bash -c 'LD_PRELOAD=/usr/lib/umad2sim/libumad2sim.so opensm -q local -P p4.conf -f -'
```

Use `smpquery pkeys` to read the P_Key table each port actually obtained:

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

The partition table is summarized as follows:

| Port              | P_Key table content | Membership in the default partition | Membership in the PartC partition |
| ----------------- | ------------------- | ----------------------------------- | --------------------------------- |
| HcaA (storage)    | `0x7fff` `0x8030`   | Limited member                      | Full member                       |
| HcaB (client)     | `0x7fff` `0x0030`   | Limited member                      | Limited member                    |
| HcaC (client)     | `0x7fff` `0x0030`   | Limited member                      | Limited member                    |
| HcaD (non-client) | `0x7fff`            | Limited member                      | None                              |

---

## 15.7 Comparison with Ethernet VLAN

Finally, let's put IB partitions side by side with the most familiar VLAN, to see clearly how they are alike in spirit yet different in form.

| Dimension              | Ethernet VLAN                                | InfiniBand Partition (P_Key)                          |
| ---------------------- | -------------------------------------------- | ----------------------------------------------------- |
| **Design intent**      | Isolate broadcast domains (suppress broadcast, shrink fault domains) | **Control reachability relationships** (closer to ACL / SAN Zoning) |
| **Identifier**         | 12-bit VLAN ID (4094 usable)                 | 16-bit P_Key (15-bit key value + 1 membership bit)    |
| **Carrying location**  | 802.1Q tag (link layer, frame header)        | P_Key field (**transport layer**, inside the BTH)     |
| **Who partitions**     | Network admin manually/by protocol on switch ports | **Subnet Manager computes and pushes down centrally** (`partitions.conf`) |
| **Isolation granularity** | Full layer-2 connectivity within the same VLAN | Within the same partition, the membership bit can slice once more (full / limited) |
| **Enforcer**           | Mainly on switch ports                       | **End HCA enforces mandatorily**, switch optional     |
| **Configuration paradigm** | Distributed: configure device by device | Centralized: one policy file, pushed uniformly by the SM |
| **"Full connectivity" default state** | Default VLAN 1                | Default partition `0x7fff` / `0xffff`                 |

The two differences most worth a network engineer's attention:

1. **Different carrying layers.** The VLAN tag is in the link-layer frame header, recognized by switches; the P_Key is in the transport-layer BTH, ultimately recognized by the end NIC.

2. **Different management paradigms.** VLAN is a product of distributed configuration: you have to configure it on each switch and also worry about trunks and VLAN passthrough; IB partitions are centralized—you only need to write one `partitions.conf`, and the rest—"which port should have which P_Keys, and how to push them down"—is all handled by the SM.

---

## 15.8 Summary

**The P_Key is IB's partition vehicle**: of the 16 bits, the low 15 bits are the partition key value (deciding "whether it's the same partition"), and the highest bit is the membership bit (deciding "whether the identity in the partition is full or limited"). Each port holds a P_Key table recording which partitions it belongs to; each data packet carries the P_Key in the BTH, and the receiver compares against it to decide whether to receive or drop.

**The default partition (`0x7fff` / `0xffff`) is the fallback for the management plane**: OpenSM automatically retains the default partition's limited membership for ports divided into tenant partitions, keeping the SM always reachable while not breaking the data isolation between tenants.
