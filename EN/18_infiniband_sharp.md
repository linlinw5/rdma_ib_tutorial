# Chapter 18: InfiniBand Fabric In-Network Computing (SHARP)

## 18.1 Starting from the "Transport Cost" of AllReduce

In the previous chapters we kept circling around one thing: how to make traffic travel better in the fabric—more balanced (adaptive routing), more isolated (partitions), more orderly (QoS), and uncongested (CC). But they share one common premise: **the network's job is always to "move data from A to B."**

The SHARP introduced in this chapter steps outside this premise. It asks a more radical question: **can the network not only "move" data, but also "compute" the data along the way?**

The origin of this question is, again, that old acquaintance from AI training: **AllReduce**.

Let me explain what AllReduce does: thousands of GPUs each compute a gradient, and all of their gradients need to be **summed**, then this sum distributed back to every GPU. The traditional approach (such as ring-allreduce) passes the data segment by segment between GPUs, adding as it passes. The problem is:

- **The same gradient data has to traverse the network back and forth many times.** The more GPUs, the larger the total transport volume;
- The adaptive routing (AR) of Chapter 14 can spread this traffic evenly and avoid hot spots, but it **cannot change "the total volume of data being moved"**—what needs to be moved still has to be moved.

So someone thought: since these gradients ultimately just need to be "added up," and they have to pass through the switches anyway, **can the switches add them up midway during forwarding**? This way, what flows in the network is no longer individual raw gradients, but "partial results" that have been summed level by level and grown smaller and smaller.

This is the core idea of **SHARP (Scalable Hierarchical Aggregation and Reduction Protocol)**, and the representative of what the industry commonly calls **In-Network Computing**.

---

## 18.2 SHARP: Moving "Reduction" into the Switch

What SHARP does, in one sentence: **offload the reduction operation in collective communication from the GPU to be completed in the switch hardware.**

Its vehicle is a **SAT (SHARP Aggregation Tree)**. SHARP-capable switches (the NVIDIA Quantum series) have a hardware reduce engine inside, called an **AN (Aggregation Node)**; these ANs are organized into a logical tree on top of the fabric:

```
                 ┌─────────┐
                 │  Root   │   ← root: computes the final result, then distributes it back down the tree
                 │   AN    │
                 └────┬────┘
            ┌─────────┴─────────┐
       ┌────┴────┐         ┌────┴────┐
       │ Switch  │         │ Switch  │   ← intermediate switches: sum the children's data and send up
       │   AN    │         │   AN    │
       └──┬───┬──┘         └──┬───┬──┘
        ┌─┘   └─┐           ┌─┘   └─┐
       GPU     GPU         GPU     GPU    ← leaves: send local gradients up the tree
```

The process of one AllReduce on SHARP becomes "one up and one down," two passes:

1. **Reduce-up (upward aggregation)**: each GPU sends its local gradient to the leaf switch it belongs to; each switch's AN **sums the data of all its children** and sends only this "partial sum" up to the next level; aggregating level by level, until the root computes the **global sum**.
2. **Broadcast-down (downward distribution)**: the root broadcasts this global result down the tree level by level, and ultimately every GPU obtains the same final result.

Compared with traditional ring-allreduce, the difference is fundamental:

|                       | Traditional ring-allreduce      | SHARP in-network reduction        |
| --------------------- | ------------------------------- | --------------------------------- |
| Who computes the reduction | GPUs pass to each other, adding as they pass | **The AN hardware in the switch computes it** |
| Data transport volume | The same data traverses the net back and forth many times | **One up and one down, two passes suffice** |
| Latency vs scale      | Grows approximately linearly with node count | **Grows logarithmically with tree height**, more predictable |
| GPU occupancy         | Must participate in every step of send/receive and summing | **Freed from the reduce computation** |

Besides summing, SHARP also supports various reduction operators such as min/max, as well as collective operations like barrier and broadcast. The new generation (from SHARPv2, paired with Quantum-2/NDR) also introduces streaming aggregation for large blocks of data, floating-point support, and so on, extending the scope of applicability from "latency-sensitive reduction of small messages" to "bandwidth-intensive reduction of large gradients."

> Adaptive Routing is about "spreading the AllReduce traffic evenly," while SHARP is about "moving less, or even not moving, this traffic at the root." One optimizes the transport path, the other reduces the transport itself. In ultra-large-scale training, the two are often used in combination.

---

## 18.3 SHARP's Components: Who Builds the Tree, Who Computes

Similar to CC, SHARP is also not something OpenSM can handle alone; it has its own set of software and hardware components:

- **Aggregation Node (AN)**: the hardware reduce engine in the switch, the place that actually "does the addition," dependent on NVIDIA Quantum-series switches.
- **Aggregation Manager (AM)**: an independent software entity (usually deployed with the SHARP software stack or UFM). It is responsible for **discovering SHARP-capable switches, building and managing the aggregation tree, and allocating tree resources**. It coexists with OpenSM but has independent duties: **OpenSM manages routing and the subnet, the AM manages the aggregation tree**.
- **The SHARP daemon (sharpd)**: runs on the compute nodes, acting as the bridge between the application and the SHARP fabric.
- **Application access**: NCCL calls SHARP via the CollNet path, offloading AllReduce to the network; MPI can also use SHARP to accelerate its collective operations.

So what is OpenSM responsible for here? **Merely "turning on the SHARP capability on supported switches."** The actual tree building and operation are left to the AM.

---

## 18.4 OpenSM's SHARP Configuration Options

In the OpenSM config file, there is just one switch related to SHARP:

```bash
# SHArP support
# 0: Ignore SHArP - No SHArP support
# 1: Disable SHArP - Disable SHArP on all supporting switches
# 2: Enable SHArP - Enable SHArP on all supporting switches
sharp_enabled 2
```

A few points worth emphasizing:

1. **OpenSM is just a "master switch" when it comes to SHARP.** All it does is "enable/disable the capability on hardware-supported switches"; as for how the aggregation tree is built and how the reduction runs, that's all on the AM and the switch hardware side.
2. **Note that repeatedly appearing "on all supporting switches."** It only takes effect on **switches that have declared SHARP capability**.

---

## 18.5 Summary

**SHARP turns the network from a "transporter" into a "participant."** Its core is offloading the heaviest collective communication in AI training (AllReduce's reduction (summing) operation) from the GPU to be completed in the switch hardware.

**The mechanism is "one up and one down" on an aggregation tree**: the leaf GPUs send gradients up the tree, each switch's Aggregation Node (AN) sums the children's data level by level and sends up only the partial result, and after the root computes the global sum it distributes it back down the tree. The result is: data transport volume drops sharply, latency grows logarithmically with tree height, and the GPU is freed from the reduce computation.

**Each component has its own role**: the AN is the hardware engine in the switch that does the addition, the AM (Aggregation Manager) is responsible for building and managing the tree, sharpd brings the application in, and OpenSM only uses a single `sharp_enabled` switch to turn the capability on for supported switches.

**Hardware feature**: it depends on NVIDIA Quantum switches and the SHARP software stack, which ibsim can't reach, so this chapter is mainly conceptual.

From Partition, QoS, and CC to SHARP, IB has step by step sunk more and more "intelligence" into the network itself. **And SHARP goes the furthest: it makes the network no longer merely a channel for computation, but a part of the computation.** This is precisely one of the key reasons InfiniBand is hard to replace easily in large-model training scenarios. The direction of "in-network computing" is also a goal the Ethernet camp is still striving to catch up with.
