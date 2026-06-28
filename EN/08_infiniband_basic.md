# Chapter 8: InfiniBand Fundamentals

Having read this far, we already understand the basic networking behavior of RDMA. Next, we enter the second part and look at RDMA's native transport channel: InfiniBand.

## 8.1 Why Learn InfiniBand Specifically

RDMA's set of semantics (Queue Pair, Verbs, one-sided operations, memory registration) was originally designed for InfiniBand in the first place. What RoCE does is essentially "move IB's transport-layer semantics onto Ethernet." In other words, all the RDMA concepts covered in the previous seven chapters have their origin in IB. RoCE is the port; IB is the original.

Only by fully understanding the architecture of the IB network can we answer several questions that can't be clearly explained in the RoCE world: Why does RoCE go to such great lengths to build a lossless network (PFC, ECN, DCQCN)? Why doesn't IB need any of these yet is inherently lossless? Why can IB's latency stably reach the microsecond level while RoCE always needs tuning? The answers to these questions are all hidden in IB's design philosophy. Only by understanding the original can we see clearly where the port's compromises lie and where it will go in the future.

This chapter focuses on just two things:

- Understanding IB's design philosophy that distinguishes it from Ethernet;
- Demystifying that pile of headache-inducing specialized terms in IB literature.

---

## 8.2 What Is InfiniBand

InfiniBand is an open-standard interconnect protocol developed by the InfiniBand Trade Association (IBTA), whose members include NVIDIA, Intel, IBM, HPE, Oracle, and others. It got started around 2002, and its goal was clear from the very beginning: to provide high-throughput, low-latency, scalable server interconnect for high-performance computing and data centers. Today it is the mainstream interconnect solution for AI training clusters and HPC.

But these descriptions don't help much in understanding its essence. The real key is one sentence:

**InfiniBand is a network architecture, vertically integrated from the physical layer to the transport layer, born specifically for RDMA.**

The Ethernet + TCP/IP we are familiar with evolved in layers, each independent of the others. Ethernet handles layer 2, IP handles layer 3, TCP handles transport; each layer was pieced together by different standards in different eras, with no unified design among them. InfiniBand is different: from the bottom link to the upper transport, it is a single unified specification, with all layers designed in coordination around one goal: "moving data across the network with minimal CPU involvement and the lowest latency." This vertical integration is precisely the root of all the differences between it and the Ethernet world.

---

## 8.3 The Fundamental Differences Between InfiniBand and Ethernet

When we put IB side by side with the Ethernet/IP world we are familiar with, several of the most crucial design differences immediately surface.

**First, lossless vs. best-effort.**

Ethernet's design philosophy is "best-effort": when a switch's buffer is full, it simply drops packets, relying on the upper-layer TCP to detect the loss, retransmit, and slow down. Packet loss is the norm, and reliability is patched in by the upper layer.

InfiniBand's link layer is inherently lossless. It uses a mechanism called **credit-based flow control**: the receiving end explicitly tells the sending end "how much buffer I still have to receive," and the sending end sends data only when the other side has space. When the buffer is insufficient, the sender stops and waits rather than dropping the packet. So in a normally operating IB network, there is simply no such thing as dropping packets due to congestion.

This directly explains RoCE's core pain point: RoCE wants to run IB's transport semantics over Ethernet, but Ethernet drops packets. So RoCE has no choice but to introduce mechanisms like PFC and ECN, going to great lengths to "retrofit" Ethernet into being lossless—and this is precisely the capability IB has innately, without needing any extra configuration. Those flow-control parameters we've tuned in RoCE deployments are essentially all patching in the lossless capability that IB gives away for free.

**Second, centralized management vs. distributed autonomy.**

Ethernet is decentralized: each switch learns MAC addresses itself, runs spanning tree itself, makes decisions itself, with no global brain. This is very flexible, but it also means no one has a global view of the entire network.

InfiniBand is centrally managed. Each IB subnet has a **Subnet Manager (SM)**, which is responsible for discovering the entire network's topology, assigning an address to each port, computing all forwarding paths, and then pushing the forwarding tables down to each switch. The entire network is uniformly planned by one central brain. This brings plug-and-play (a node is automatically discovered and configured as soon as it's connected) and globally optimal routing, at the cost of needing such a management entity to exist.

Ethernet, in its later development, also talks about "auto-discovery, auto-deployment"—you often hear these terms in Cisco ACI, DNAC, and the like—but most of them rely on a set of external controllers. Yet "auto-discovery, auto-deployment" is a capability IB has innately.

**Third, link-layer addressing vs. IP addressing.**

In the Ethernet + TCP/IP world, cross-network communication relies on IP addresses and hop-by-hop routing; InfiniBand is completely different, using a local identifier called the LID to complete forwarding directly at the link layer.

In TCP/IP we often say "layer 2 switches, layer 3 routes." Ethernet's layer-2 forwarding rules are simple and crude—nothing more than the "flood, learn, forward, age" MAC address table mechanism; the real path selection doesn't appear until the network layer (layer 3), giving rise to various routing algorithms like RIP, EIGRP, OSPF, and BGP, which let packets pick out an optimal path hop by hop. In other words, TCP/IP routing solves communication between subnets, while within a subnet it's simple and crude layer-2 switching.

The InfiniBand world is completely different: the link layer shoulders both the "switching" and "routing" duties—it must quickly deliver data to the correct port, and it must also complete routing/addressing (the forwarding table of each IB switch is pre-computed by the SM using routing algorithms). In other words, path computation has already happened at the link layer, rather than relying on the "brute-force" forwarding of broadcast and flooding. As for cross-subnet communication, that's nothing more than a matter of "making the network bigger"; and in reality, IB networks rarely need multi-subnet interconnect, since the scale of a single subnet is usually already sufficient to cover the vast majority of scenarios.

**Fourth, hardware transport layer vs. software protocol stack.**

In the Ethernet world, the TCP/IP protocol stack runs in kernel software, executed by the CPU. InfiniBand builds the entire transport layer (segmentation, reassembly, acknowledgment, retransmission) into the NIC hardware. This is precisely the physical basis on which RDMA can bypass the kernel and bypass the CPU: protocol processing was never on the CPU to begin with, but inside the NIC's chip.

Putting these four points together, IB's image becomes clear: it is a network uniformly scheduled by a central brain, with the link layer carrying its own lossless guarantee, and with addressing and protocol processing both pushed down into hardware. It is not "faster Ethernet" but a different paradigm altogether.

---

## 8.4 Demystifying the Physical-Layer Rate Codenames: How Many G Is SDR Through XDR

The most off-putting thing for newcomers in IB literature is the pile of codenames SDR, QDR, HDR, NDR—you can't tell the rate from the name at all, and you have to look up a table every time. This section explains it thoroughly.

First, understand one most crucial concept: **the bandwidth of an IB port = the rate of a single lane × the number of lanes.**

IB uses multiple physical channels (lanes) transmitting in parallel to stack up bandwidth; common port widths are 1x, 4x, and 12x. **The vast majority of server NICs, and the rates quoted in literature, default to 4x.** This is the biggest source of the table mismatch: for the same codename NDR, a single lane is 100G, but the "NDR 400G" we usually hear refers to 4x (4 lanes × 100G). So when reading literature, you must distinguish whether it's talking about per-lane or per-port.

In the table below, per-lane is the baseline defined by the specification, and the 4x column is the port rate you'll actually deal with most often:

| Codename | Full name          | Per-lane rate | 4x port rate | Encoding | Starting era |
| -------- | ------------------ | ------------- | ------------ | -------- | ------------ |
| SDR      | Single Data Rate   | 2.5 Gb/s      | 10 Gb/s      | 8b/10b   | 2001         |
| DDR      | Double Data Rate   | 5 Gb/s        | 20 Gb/s      | 8b/10b   | —            |
| QDR      | Quad Data Rate     | 10 Gb/s       | 40 Gb/s      | 8b/10b   | 2007         |
| FDR      | Fourteen Data Rate | 14.0625 Gb/s  | 56 Gb/s      | 64b/66b  | 2011         |
| EDR      | Enhanced Data Rate | 25 Gb/s       | 100 Gb/s     | 64b/66b  | 2014         |
| HDR      | High Data Rate     | 50 Gb/s       | 200 Gb/s     | 64b/66b  | 2018         |
| NDR      | Next Data Rate     | 100 Gb/s      | 400 Gb/s     | 64b/66b  | 2022         |
| XDR      | (next) Data Rate   | 200 Gb/s      | 800 Gb/s     | 64b/66b  | 2023+        |

A few pitfalls to watch for when reading the table:

**The naming follows no discernible pattern, so don't try to memorize rules.** SDR/DDR/QDR could still be read as "single/double/quad" in the early days, but after FDR (Fourteen, referring to 14G) it's purely marketing naming—F, E, H, N, X are neither abbreviations nor a sequence, and can only be memorized by rote or looked up.

**The nominal rate is not equal to the effective rate; the difference lies in encoding overhead.** What travels on the physical line is not just data but also redundant bits used for clock synchronization and error correction. The three generations SDR/DDR/QDR use 8b/10b encoding: out of every 10 bits, only 8 are effective data, with overhead as high as 20%. So QDR 4x is nominally 40G, but the actual effective throughput is only about 32G. Starting from FDR, the switch to 64b/66b encoding drops the overhead sharply to about 3%, much more efficient. When you see "theoretical bandwidth" not matching "actual bandwidth," this is the reason.

**XDR and beyond.** XDR's specification was published by the IBTA in 2023, with a single lane at 200G and a 4x port at 800G, currently used mainly in the latest GPU clusters. The IBTA roadmap also has higher generations in planning, with rates still continuing to double.

In practice there's a mental conversion trick: **when you hear a codename, default to assuming it's 4x, then look up the 4x column in the table above**, and you'll basically not go wrong. When you need precision, go back and confirm whether it's per-lane or 12x.

---

## 8.5 Core Concepts: Addresses and Identifiers

There are three most basic identifiers in IB: GUID, LID, and GID, which beginners very easily confuse. An analogy makes it clear:

> The GUID is like a device's MAC address (burned in at the factory, never changes), the LID is like a temporarily assigned street number within a subnet, and the GID is like a globally identifying address.

**GUID (Globally Unique Identifier)**

A 64-bit permanent identifier burned in by the vendor when each IB device (NIC, switch, port) leaves the factory, globally unique, and unchanged even after a power-off reboot. It identifies "who this hardware is," similar to Ethernet's MAC address. Common ones are the Node GUID (identifying the whole device) and the Port GUID (identifying a specific port).

**LID (Local Identifier)**

A 16-bit address **dynamically assigned** by the Subnet Manager when a device connects, valid only within the current subnet. Link-layer forwarding within an IB subnet relies precisely on the LID; a switch looks at the destination LID in the packet header to decide which port to send to. Because it is 16 bits, an IB subnet can theoretically have at most about 48,000 LID addresses, which is exactly where the figure "a single subnet has at most about 48,000 nodes" comes from.

**GID (Global Identifier)**

A 128-bit global address, in the same format as IPv6, used for **cross-subnet** communication. When data needs to leave its own subnet and pass through an IB router to reach another subnet, it relies on the GID for addressing. The GID consists of two parts: the subnet prefix (identifying which subnet) + the interface identifier (usually derived from the Port GUID).

The relationship among the three can be strung together like this: the GUID is the hardware's permanent identity, the LID is the temporary street number it gets in the local subnet, and the GID is the global address it uses when going across subnets. Intra-subnet communication only needs the LID; only cross-subnet communication brings in the GID.

Here I want to make a special point: do not use the layer-2/layer-3 address mapping (ARP) and "onion-style" encapsulation/decapsulation concepts from the TCP/IP world to understand IB; there are no corresponding concepts in the IB world.

We'll introduce IB's protocol stack in Chapter 10; here you only need a basic understanding of these concepts.

---

## 8.6 IB's Management Plane: The SM, SMA, SA, MAD Family

IB is centrally managed, and at its core is the Subnet Manager. But there is a ring of terms around it: SM, SMA, SA, MAD, SMP, GMP. These are unavoidable when reading IB literature, so let's put them together and clarify their relationships.

**SM (Subnet Manager)** is the brain of the entire subnet. It does four things: discover the topology (which devices are in the network and how they're connected), assign a LID to each port, compute the forwarding paths, and push the forwarding tables down to each switch. A subnet can have multiple SMs, but only one is the master, with the rest on standby; if the master fails, a standby takes over—guaranteeing no single point of failure in the management plane.

**SMA (Subnet Management Agent)** is a small agent built into each IB device. The SM is the brain, and the SMA is the "liaison" the brain places on each node. Whenever the SM wants to query a device's state or configure its LID, it does so by communicating with that device's SMA.

**MAD (Management Datagram)** is the standard message format used for management-plane communication. All conversations between the SM and the SMA (queries, configuration, reporting) are encapsulated as MADs to be conveyed. It can be understood as IB's management-plane "universal envelope."

MADs are divided into two categories by purpose:

- **SMP (Subnet Management Packet)**: MADs used specifically for subnet management—for example, the SM discovering the topology, assigning LIDs, and configuring ports all use SMPs. The SMP has a special capability: it can work even before the LID has been assigned (traveling hop by hop along a path via a method called directed routing), which is how the SM can complete initial discovery in the "chicken-and-egg" stage when the network has just powered on and no one has an address yet.
- **GMP (General Management Packet)**: used for management tasks other than subnet management, such as performance monitoring and device information queries.

**SA (Subnet Administrator)** is the query service interface the SM provides externally. The SM holds the whole network's information (who has what LID, how paths go); when an ordinary node wants to know "to communicate with a certain node, which path should I take and what parameters should I use," it sends a query request to the SA. Therefore, the SA is an "information desk" that holds the whole network's path information and is available for everyone to query.

Stringing this family together: **the SM is the brain, relying on the SMAs spread across all nodes as liaisons, communicating with each other using the envelope of MADs (SMP for managing the subnet, GMP for everything else), while the SA is the query window the SM opens up to all nodes.**

---

## 8.7 IB Packet Structure: What's in the Headers

An IB packet, like an Ethernet packet, is formed by the payload being "sandwiched" between the headers and trailers of each layer. Below is the field layout of a complete IB data packet:

```
                        recomputed per hop, covers whole packet ──┐
  invariant end-to-end, covers only unchanging fields ─┐          │
                                                       │          │
┌──────────┬──────────┬──────────┬──────────────┬──────────┬──────────┐
│   LRH    │   GRH    │   BTH    │   Payload    │   ICRC   │   VCRC   │
│  Local   │  Global  │   Base   │     Data     │ Invariant│   Link   │
│  Routing │  Routing │ Transport│   payload    │   CRC    │   CRC    │
│  Header  │  Header  │  Header  │              │          │          │
└──────────┴──────────┴──────────┴──────────────┴──────────┴──────────┘
  8 bytes   40 bytes   12 bytes    variable       4 bytes    2 bytes
  always     inter-     always                    always     always
            subnet
             only
     │          │          │
     │          │          └─ dst QP, opcode (Send/Write/Read), PSN
     │          └─ src/dst GID, inter-subnet routing only
     └─ src/dst LID, intra-subnet forwarding

```

The role of each field, one by one:

**LRH (Local Routing Header)**: contains the source and destination LIDs and is the basis for link-layer forwarding within the subnet. A switch looks at just this header to decide which port to forward to. Every IB packet has it.

**GRH (Global Routing Header)**: contains the source and destination GIDs, and appears only when cross-subnet routing is needed. For intra-subnet communication this header is omitted.

**BTH (Base Transport Header)**: the core header of the transport layer, containing the destination QP number, the operation type (whether it's Send, RDMA Write, or Read), and the PSN (Packet Sequence Number). Those QPNs and PSNs exchanged during connection setup are ultimately filled into this header; every IB packet has it.

**Payload**: the data actually transferred, of variable length.

**ICRC (Invariant CRC)**: covers the fields in the packet that don't change during end-to-end transmission, performing an end-to-end integrity check.

**VCRC (Variant CRC)**: covers the entire packet, is recomputed at every hop, and performs a link-level integrity check.

The division of labor between ICRC and VCRC is an interesting design: some fields (such as parts that may be modified during forwarding) change hop by hop, while some fields are constant end to end. The VCRC, recomputed at each hop, guarantees that the link transmission had no errors, while the ICRC, invariant end to end, guarantees that the core data was not corrupted from start to finish. The two layers of checks each cover their own segment.

---

## 8.8 RDMA's Software Stack: Kernel Drivers, rdma-core, and MLNX_OFED

The preferred platform for RDMA development is Linux. In this section we discuss how the RDMA software stack on the Linux platform divides its labor.

To understand all this, first remember one most crucial fact: **the RDMA software stack is split into two layers, kernel space and user space.**

---

### Kernel Space: The Drivers Have Long Been in the Mainline Kernel

The RDMA drivers in the true sense are a set of kernel modules, released together with the Linux mainline kernel; the moment we finish installing Ubuntu they're already there, with no need to install them separately. Common ones include:

- `ib_core`: the core framework of the RDMA subsystem, the common foundation for all RDMA devices.
- `mlx5_core` / `mlx5_ib`: the drivers for NVIDIA/Mellanox ConnectX-series NICs.
- `rdma_rxe`: the kernel implementation of Soft-RoCE, which lets ordinary Ethernet NICs without an RDMA NIC run RDMA in software simulation; this is exactly what we used when tinkering with Soft-RoCE earlier.

This means: on a freshly installed Ubuntu, the kernel-side RDMA capability is ready-made, and all that's missing are the upper-layer libraries and tools.

---

### User Space: rdma-core Provides the Libraries and Tools

The `rdma-core` package provides the **user-space** part, that is, the bridge between applications and the kernel drivers:

- `libibverbs`: the Verbs library; the API that RDMA programming calls comes from here.
- `librdmacm`: the RDMA CM library, responsible for connection management.
- The user-space provider libraries for each NIC (such as `libmlx5`), responsible for connecting generic Verbs calls to specific hardware.
- A set of diagnostic tools: `ibv_devinfo`, `ibstat`, `rdma`, `ibping`, etc., used to observe information such as LID, GID, port state, and the Subnet Manager.

Installing rdma-core on Ubuntu:

```bash
apt install rdma-core
```

In summary: **the mainline kernel provides the RDMA drivers (kernel space), `rdma-core` adds the Verbs library, the CM library, and the diagnostic tools (user space), and only together do the two make a usable open-source RDMA software stack.** This "mainline kernel + rdma-core" combination is precisely the contemporary form into which the historical standalone mega-package OFED evolved: the user space converged into `rdma-core`, the kernel space was merged into the mainline, and distributions work out of the box.

---

### MLNX_OFED

Since the open-source stack works out of the box, what is NVIDIA's **MLNX_OFED** for?

MLNX_OFED is an **entire replacement solution** that NVIDIA packages for its own NICs. It does not patch the open-source stack; instead, it uses a whole set of components compiled by NVIDIA itself to **wholesale supplant** the open-source components mentioned above:

- Kernel space: replace the version bundled with the mainline kernel with NVIDIA's own `mlx5` module.
- User space: replace `rdma-core` with NVIDIA's own `libibverbs`, `libmlx5`, etc.

During installation it uninstalls or overrides the system's existing `rdma-core`, after which what runs on the system is this whole NVIDIA set, rather than the distribution's mainline one.

So when do you actually need it? Roughly in these situations:

- You're using NVIDIA/Mellanox NICs and want the official performance optimizations, the latest features, or vendor technical support.
- You're running on an older distribution/kernel and need the compatibility support NVIDIA provides.
- You want to use advanced features like GPUDirect RDMA that require a matching driver stack.

Conversely, if you're just learning, experimenting, or running a lab with Soft-RoCE as in the previous chapters, you don't need MLNX_OFED at all. The mainline kernel + `rdma-core` is enough; they're actually cleaner, naturally stay in sync with the kernel version, and spare you from worrying about module/kernel mismatches.
