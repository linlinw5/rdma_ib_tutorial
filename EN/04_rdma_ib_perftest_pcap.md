# Chapter 4: Packet Capture Analysis of the RDMA IB perftest Tool

Starting from this chapter, we use RDMA's standard tools to initiate RDMA communication between two devices, then capture the packets for analysis.

The lab environment currently has two VMs in total, with Soft-RoCE installed. For the subsequent tests, `10.1.16.61` will serve as the server and `10.1.16.62` as the client.

> Since I don't have IB hardware on hand, I can only observe RDMA through `Ethernet + Soft-RoCE`. Although the performance metrics can't match those in reality, the upside is that anyone can use the same method to build an identical lab environment on their own computer/server, with no special hardware requirements whatsoever.

## 4.1 A Brief Introduction to the perftest Tool

perftest is a dedicated performance testing tool with complete functionality. It is the industry-standard RDMA performance benchmark suite, and the bandwidth numbers in the datasheets of NIC vendors (Mellanox/NVIDIA, Intel) are basically all produced with perftest.

Grouped by operation type:

```
Bandwidth tests (_bw):
  ib_write_bw      RDMA Write bandwidth
  ib_read_bw       RDMA Read bandwidth
  ib_send_bw       RDMA Send/Recv bandwidth
  ib_atomic_bw     RDMA Atomic bandwidth

Latency tests (_lat):
  ib_write_lat     RDMA Write latency
  ib_read_lat      RDMA Read latency
  ib_send_lat      RDMA Send/Recv latency
  ib_atomic_lat    RDMA Atomic latency
```

```bash
# View the version and the complete tool list
expert@k8s-61:~$ dpkg -L perftest | grep bin
/usr/bin
/usr/bin/ib_atomic_bw
/usr/bin/ib_atomic_lat
/usr/bin/ib_read_bw
/usr/bin/ib_read_lat
/usr/bin/ib_send_bw
/usr/bin/ib_send_lat
/usr/bin/ib_write_bw
/usr/bin/ib_write_lat
/usr/bin/raw_ethernet_burst_lat
/usr/bin/raw_ethernet_bw
/usr/bin/raw_ethernet_fs_rate
/usr/bin/raw_ethernet_lat
/usr/bin/run_perftest_loopback
/usr/bin/run_perftest_multi_devices
```

In this chapter we will perform packet capture analysis on `ib_write_bw`, `ib_read_bw`, `ib_send_bw`, `ib_write_lat`, `ib_read_lat`, and `ib_atomic_bw`, in order to understand the RDMA communication process in depth.

---

## 4.2 Packet Capture Analysis of ib_write_bw

[pcap capture file](../pcap/rdma_write_bw.pcap)

### Test Environment Setup

To make the data easier to observe, we deliberately specify the size of each write as 3000 bytes (`-s 3000`) when invoking the command, and run a total of 5 iterations (`-n 5`).

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/rdma_write_bw.pcap 'port 18515 or udp port 4791'

# Server
ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000

# Client
ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000 10.1.16.61
```

```bash
expert@k8s-61:~$ ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 CQ Moderation   : 5
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x003d PSN 0xf6cebf RKey 0x003284 VAddr 0x00605866c0b000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
 remote address: LID 0000 QPN 0x003f PSN 0x68fbff RKey 0x00368f VAddr 0x0063f7fb80b000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 3000       5           0.084385            0.084385            0.003516
---------------------------------------------------------------------------------------




expert@k8s-62:~$ ib_write_bw -d rxe0 --report_gbits -n 5 -s 3000 10.1.16.61
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 TX depth        : 5
 CQ Moderation   : 5
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x003f PSN 0x68fbff RKey 0x00368f VAddr 0x0063f7fb80b000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
 remote address: LID 0000 QPN 0x003d PSN 0xf6cebf RKey 0x003284 VAddr 0x00605866c0b000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 3000       5           0.084385            0.084385            0.003516
---------------------------------------------------------------------------------------
```

After the test finishes, press Ctrl+C to stop the capture, copy the pcap to your desktop, and open it with Wireshark.

![ib_write_bw1](../assets/ib_write_bw_1.png)

From the captured data, we can see that `10.1.16.62` initiates a connection to `10.1.16.61`, and the data consists of two parts: TCP 18515 and UDP 4791. From the overall behavior we can roughly infer the workflow of perftest's ib_write_bw:

- First, use TCP 18515 to complete the bootstrap, with both sides advertising to each other their QPN, PSN, Virtual Address, rkey, and so on, preparing for RDMA transfer;
- Begin RDMA write, with the QP type being RC and the port being UDP 4791;
- Tear down the TCP connection.

> RRoCE = Routable RoCE, which is just another name for RoCEv2.
>
> Since the bootstrap phase uses TCP, which is a private implementation of the perftest tool, Wireshark cannot parse the payload content within the TCP, which is normal.

Next, we'll expand the InfiniBand header of the first RoCE packet in the data and take a look at the information in its two headers, the BTH and the RETH.

### Interpreting the BTH Fields

```
Base Transport Header (BTH)
  Opcode:                  0x06  RC - RDMA WRITE First
  Solicited Event (SE):    0     ← does not request the peer to trigger a CQ event notification
  MigReq:                  0     ← currently on the primary path, not the migration path
  Pad Count:               0     ← payload needs no padding bytes (already aligned)
  Header Version:          0     ← fixed at 0 by the IB spec
  Partition Key (PKey):    0xFFFF← decimal 65535, the default global PKey, indicating unrestricted partition membership (covered in the InfiniBand chapter)
  Destination QP:          0x00003d ← the peer's QP number
  Acknowledge Request (A): 0     ← does not require the peer to reply with an ACK immediately
  Packet Sequence Number:  6880255 ← the sequence number of the current packet, used for ordering and loss detection
```

**Field-by-field explanation:**

**Opcode = RC RDMA WRITE First (6)**: the opcode simultaneously encodes the QP type (RC) and the packet's position within the message (First). It is precisely because it carries the First marker that this packet carries a RETH; the subsequent MIDDLE/LAST packets have different opcodes and therefore do not carry a RETH.

**Solicited Event = 0**: when the SE bit is set to 1, the peer's RC layer will post a Solicited event to the CQ after receiving the packet, triggering a waiting `ibv_get_cq_event()`. RDMA Write is a one-sided operation that the peer's CPU is unaware of, so SE is usually set to 0.

**MigReq = 0**: IB supports APM (Automatic Path Migration), where a QP can have a primary/backup pair of paths preconfigured; MigReq=1 indicates a request for the peer to switch to the backup path. Soft-RoCE does not implement APM, so it is fixed at 0.

**Pad Count = 0**: the IB payload requires 4-byte alignment; if the actual data falls short, Pad Count bytes are padded at the end of the payload, and these padding bytes must be excluded when computing the ICRC. This packet's 1024 bytes are exactly aligned, so no padding is needed.

**PKey = 0xFFFF**: the Partition Key is used for IB's subnet partition isolation (similar to ACLs in an IP network or Zones in a SAN network); 0xFFFF is the Full Member default value, indicating that the packet belongs to the default partition, and any QP that is also in the default partition can receive it.

**Destination QP**: the BTH contains only the destination QPN, not the source QPN (the source QPN appears only in the DETH of the UD type). When an RC connection is established, the two parties have already exchanged QPNs, so the BTH only needs to record the destination one-directionally.

**Acknowledge Request = 0**: when the A bit is set to 1, it forces the peer to reply with an ACK immediately upon receiving the packet. When A=0, the peer can accumulate multiple packets and then reply with a single unified ACK (cumulative acknowledgment), reducing the number of ACK packets and improving throughput.

**PSN**: the sender maintains a monotonically increasing PSN counter. The receiver uses the PSN to detect reordering and packet loss—if the received PSN is not contiguous, a NAK is triggered. The PSNs of the FIRST/MIDDLE/LAST packets of the same message increment contiguously, and they also remain globally monotonic even when interleaved with packets of other messages.

---

### Interpreting the RETH Fields and the Fragmentation Principle

Since this test specifies a size of 3000 bytes, while Soft-RoCE's default MTU is 1024 bytes, each RDMA Write becomes 3 IB packets:

```
Packet 1  RDMA_WRITE_FIRST   1024 bytes  carries RETH (VAddr + RKey + DMALen=3000)
Packet 2  RDMA_WRITE_MIDDLE  1024 bytes  no RETH
Packet 3  RDMA_WRITE_LAST     952 bytes  no RETH (3000 - 1024 - 1024 = 952)
```

```
RETH - RDMA Extended Transport Header
  Virtual Address:  0x0000605866c0b000  ← the starting virtual address of the registered memory on the target side
  Remote Key:       0x00003284          ← the authorization token, by which the sender accesses the peer's memory
  DMA Length:       3000 (0x00000bb8)   ← the total number of bytes for this Write
```

**Virtual Address**: this is the virtual address of the Responder's (receiver's) MR, communicated by the Responder to the Initiator out-of-band before connection setup (exchanged after `ibv_reg_mr`). Once the NIC obtains this address, it directly uses the RKey for permission verification, then performs the DMA write to the corresponding physical pages.

**RKey**: the Remote Key is the access token returned by `ibv_reg_mr`, bound to a specific memory range and access permissions (Read/Write/Atomic). After the Responder's RDMA engine receives the packet, it first verifies whether the RKey is valid, then performs the DMA write. The RKey can be invalidated, expiring after one-time use (used in security-sensitive scenarios).

**DMA Length = 3000**: the RETH appears only in the FIRST packet, but it carries the total length of the entire message. This allows the Responder to pre-allocate the buffer and verify the MR boundary upon receiving the first packet. The subsequent MIDDLE/LAST packets carry no RETH; the Responder relies on PSN contiguity and the DMA Length to deduce the offset position at which each packet is written.

---

Now let's look at the RDMA ACK packet. This is an RC-type QP, so every time `10.1.16.62` completes a whole write to `10.1.16.61`, `10.1.16.61` must respond with an ACK to `10.1.16.62`.

![ib_write_bw2](../assets/ib_write_bw_2.png)

### Interpreting the ACK AETH Fields

```
AETH - ACK Extended Transport Header
  Syndrome: 31, Ack
    Reserved:     0
    OpCode:       Ack (0)      ← this is a normal ACK, not a NAK
    Credit Count: 31           ← 31 credits remaining
  Message Sequence Number: 1  ← this is the response to message #1
```

**OpCode = Ack** indicates that the data is normal; if it were a NAK, it would trigger a retransmission.

**Credit Count** is an end-to-end receive credit hint at the RC transport layer (End-to-End Receive Credit Hint), used to tell the peer how many more requests requiring Receive Queue resources it can currently accept, thereby reducing the occurrence of RNR NAK (Receiver Not Ready Negative Acknowledgement).

> AETH Credit has no flow-control significance whatsoever for the data transfer of RDMA Read and RDMA Write. Its design purpose is precisely to provide an end-to-end receive credit hint for operations that need to consume Receive Queue resources (mainly Send-type operations).
>
> The rxe driver does not fully implement this flow-control mechanism; the field usually returns a fixed value of 31, serving only as a protocol placeholder and not reflecting the real Receive Queue state.
>
> The Credit here and the Credit-Based Flow Control of the InfiniBand link layer are two different things, with no relationship to each other.

---

## 4.3 Packet Capture Analysis of ib_read_bw

[pcap capture file](../pcap/rdma_read_bw.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/rdma_read_bw.pcap 'port 18515 or udp port 4791'

# Server
ib_read_bw -d rxe0 --report_gbits -n 5 -s 3000

# Client
ib_read_bw -d rxe0 --report_gbits -n 5 -s 3000 10.1.16.61

```

### Interpreting the Read Mechanism

![ib_read-bw1](../assets/ib_read_bw_1.png)

![ib_read-bw2](../assets/ib_read_bw_2.png)

From the capture we can see that RDMA Read involves two kinds of packets, where the **Request is very small and the Response is very large**. This is a characteristic of RDMA Read: the request is just a "fetch data" instruction, and the data is pushed back by the party being requested:

```bash

Packet 22  Read Request
      BTH: QPN=0x1c, PSN=6433910
      RETH: VAddr=peer memory address, RKey=peer key, DMALength=3000
      → the client says: go fetch the 3000 bytes at that address over there for me

# Then the server splits the 3000 bytes into 3 fragments (1024 + 1024 + 952) and pushes them back.

Packet 27  Read Response First:   BTH + AETH + Data
Packet 28  Read Response Middle:  BTH + Data
Packet 29  Read Response Last:    BTH + AETH + Data
```

The Read Response Middle does not carry an AETH, because the middle fragment is just data transport, with no new state information that needs to be conveyed.

The Read Response Last must carry an AETH, because the initiator waits for this completion signal before it can release the Outstanding Read slot.

> Note: "Outstanding" in IB literally means "pending/unresolved"; in the IB context it refers to an operation that has already been issued but has not yet received its final acknowledgment.

---

## 4.4 Packet Capture Analysis of ib_send_bw

![ib_send-bw](../assets/ib_send_bw.png)

[pcap capture file](../pcap/rdma_send_bw.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/rdma_send_bw.pcap 'port 18515 or udp port 4791'

# Server
ib_send_bw -d rxe0 --report_gbits -n 5 -s 3000

# Client
ib_send_bw -d rxe0 --report_gbits -n 5 -s 3000 10.1.16.61
```

### Comparative Analysis of Header Structures (Send vs Write)

One thing worth reiterating: RDMA Send is a two-sided operation, while Write is a one-sided operation; a two-sided operation requires the involvement of the remote CPU.

Below is a comparison of the header information of Send and Write:

```
Send First:                      Write First:
  BTH                              BTH
    Opcode: RC SEND First(0)         Opcode: RC WRITE First(6)
    QPN: 0x000023                    QPN: 0x00001b
    PSN: 16733043                    PSN: 5454383
    Ack Request: False               Ack Request: False
  [no extension header]              RETH
  Data(1024)                           VAddr: 0x00005e6b89b96000
                                       RKey:  0x00000ce6
                                       DMALen: 3000
                                     Data(1024)
```

**Send has no RETH**, which is the most essential difference, because the sender simply doesn't know which address on the peer the data should be written to; that decision rests with the receiver, who decides where the data lands via the buffer specified when posting Recv.

At the network level, the two are nearly identical: both are RC, both have ACKs, both have First/Middle/Last fragmentation, and both go over UDP 4791.

The real difference is on the host side:

```
Write:  the receiver's CPU is uninvolved throughout
          the sender specifies the target address (VAddr+RKey)
          data lands directly in the peer's memory, with the peer application unaware

Send:   the receiver's CPU must be involved in advance
          must post Recv ahead of time, preparing the receive buffer
          after the data arrives, the CQ generates a completion event, and only then does the application know it received data
```

NCCL in AI clusters tends to use Write rather than Send. Because Write completely bypasses the peer's CPU, latency is lower, and the receiver doesn't need to poll the CQ to cooperate with the sender.

---

## 4.5 Packet Capture Analysis of ib_write_lat

[pcap capture file](../pcap/rdma_write_lat.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/rdma_write_lat.pcap 'port 18515 or udp port 4791'

# Server
ib_write_lat -d rxe0 -n 5 -s 64

# Client
ib_write_lat -d rxe0 -n 5 -s 64 10.1.16.61
```

The test result output is as follows:

```bash
expert@k8s-61:~$ ib_write_lat -d rxe0 -n 5 -s 64

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write Latency Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: OFF
 ibv_wr* API     : OFF
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x003e PSN 0x521172 RKey 0x0033d2 VAddr 0x0063b7da32d000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
 remote address: LID 0000 QPN 0x0040 PSN 0x1200a RKey 0x00371a VAddr 0x005fd8d3289000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
 64      5          185.81         219.13       193.75                 193.75           7.00            219.13                  219.13
---------------------------------------------------------------------------------------


expert@k8s-62:~$ ib_write_lat -d rxe0 -n 5 -s 64 10.1.16.61
---------------------------------------------------------------------------------------
                    RDMA_Write Latency Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: OFF
 ibv_wr* API     : OFF
 TX depth        : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0040 PSN 0x1200a RKey 0x00371a VAddr 0x005fd8d3289000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
 remote address: LID 0000 QPN 0x003e PSN 0x521172 RKey 0x0033d2 VAddr 0x0063b7da32d000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
 64      5          175.75         210.59       191.96                 191.96           16.00           210.59                  210.59
---------------------------------------------------------------------------------------

```

![ib_write_lat](../assets/ib_write_lat.png)

Unlike the bandwidth test, the write latency test is a strict ping-pong mode: both the server and the client actively send Writes, and one round trip counts as one round-trip latency:

```bash
Server                Client
   | ---- Write ----> |
   | <--- Ack   ----- |
   |                  |
   | <--- Write ----- |
   | ---- Ack   ----> |
```

Explanation of the result parameters and their practical use cases:

| Metric                    | Meaning                                            | Typical use case | Concern it addresses                                          |
| ------------------------- | -------------------------------------------------- | ---------------- | ------------------------------------------------------------- |
| `t_min`                   | The shortest of all iterations, representing the system's theoretical best latency. | Functional verification | Is the link working? Are there obvious configuration problems or extra overhead? |
| `t_typical`               | A typical value close to the median, trying to exclude the influence of outlier samples. | Performance baseline | What is the real latency level of most requests? |
| `t_avg`                   | The arithmetic mean of all samples.                | Performance baseline | What is the overall average performance level of the system? |
| `t_stdev`                 | The standard deviation of the latency, measuring the dispersion of the samples. | Stability assessment | Is the latency fluctuation too large? Does the system have obvious jitter? |
| `t_max`                   | The longest of all iterations, representing the worst case. | Anomaly analysis | Are there sporadic anomalies, such as CPU preemption, kernel scheduling delays, or transient network congestion? |
| `99% percentile (P99)`    | Out of 100 requests, 99 have a latency not exceeding this value. | Production SLA | Does the experience ceiling of almost all requests meet business requirements? |
| `99.9% percentile (P999)` | Out of 1000 requests, 999 have a latency not exceeding this value. | Production SLA | Does the extreme tail latency meet the requirements of real-time business and SLAs? |

> **Note: When the number of samples is small (e.g., this example runs only 5 iterations), P99 and P999 have no statistical significance, and their values usually degenerate into the maximum value among the samples.**

---

## 4.6 Packet Capture Analysis of ib_read_lat

[pcap capture file](../pcap/rdma_read_lat.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/read_lat.pcap 'port 18515 or udp port 4791'

# Server
ib_read_lat -d rxe0 -n 5 -s 64

# Client
ib_read_lat -d rxe0 -n 5 -s 64 10.1.16.61
```

From the capture, the Read LAT test is not a ping-pong mode; each round has only 2 packets: the client initiates a read request, and the server responds with a read response. This is fairly simple, so we won't do a detailed analysis.

---

## 4.7 Packet Capture Analysis of ib_atomic_bw

[pcap capture file](../pcap/rdma_atomic_bw.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/atomic_bw.pcap 'port 18515 or udp port 4791'

# Server
ib_atomic_bw -d rxe0 -n 5

# Client
ib_atomic_bw -d rxe0 -n 5 10.1.16.61

# No `-s` parameter is needed; Atomic operations are fixed at 8 bytes, and the size cannot be specified.
```

The test results are as follows:

```bash
expert@k8s-61:~$ ib_atomic_bw -d rxe0 -n 5
---------------------------------------------------------------------------------------
Device not recognized to implement inline feature. Disabling it

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    Atomic FETCH_AND_ADD BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 CQ Moderation   : 5
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x003f PSN 0x440996
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
 remote address: LID 0000 QPN 0x0041 PSN 0x163c1a
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8          5                0.06               0.06               0.008390
---------------------------------------------------------------------------------------


expert@k8s-62:~$ ib_atomic_bw -d rxe0 -n 5 10.1.16.61
---------------------------------------------------------------------------------------
Device not recognized to implement inline feature. Disabling it
---------------------------------------------------------------------------------------
                    Atomic FETCH_AND_ADD BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 TX depth        : 5
 CQ Moderation   : 5
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0041 PSN 0x163c1a
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:62
 remote address: LID 0000 QPN 0x003f PSN 0x440996
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:01:16:61
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8          5                0.06               0.06               0.008390
---------------------------------------------------------------------------------------
```

### Parsing the Fetch Add Request Packet (Packet 22)

![ib_atomic_bw](../assets/ib_atomic_bw.png)

```
BTH:
  Opcode:  RC FetchAdd (22)
  QPN:     0x00003f
  PSN:     0x163c1a
  Acknowledge Request: True        ← explicitly requires the peer to reply with an ACK

AtomicETH (Atomic Extended Transport Header, 28 bytes):
  VAddr:   0x0000619658999000      ← the target memory address
  RKey:    0x000034b7              ← the remote access key
  Add Data: 1                      ← adds 1 each time
  Compare:  0                      ← Fetch Add doesn't use this field, set to 0
```

### The Compare Data Field

The AtomicETH has two fields: `Add Data` and `Compare Data`. This is because the same header structure serves both the **Fetch Add** and **Compare and Swap** operations.

`ib_atomic_bw` uses Fetch Add by default, so Compare is set to 0.

### Parsing the Atomic Acknowledge Response Packet (Packet 23)

![ib_atomic_bw_2](../assets/ib_atomic_bw_2.png)

```
BTH:
  Opcode:  RC ATOMIC Acknowledge (18)   ← a dedicated opcode, not an ordinary ACK

AETH:
  OpCode:     Ack (0)
  Credit Count: 31

ATOMICACKETH (Atomic ACK Extended Transport Header, 8 bytes):
  Original Remote Data: 14233087857548734390  ← the old value before the operation, returned to the initiator
```
