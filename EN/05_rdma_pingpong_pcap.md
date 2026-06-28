# Chapter 5: Packet Capture Analysis of RDMA IB pingpong

Continuing the tests from the previous chapter, the lab environment remains unchanged, with `10.1.16.61` as the server and `10.1.16.62` as the client.

## 5.1 A Brief Introduction to libibverbs-utils

The libibverbs library comes with some simple example programs that contain little code, originally intended to let developers learn the basic flow of RDMA programming. In this chapter, we use these built-in testing tools to initiate RDMA communication, mainly three small pingpong programs.

> https://github.com/linux-rdma/rdma-core/tree/master/libibverbs/examples

`libibverbs` differs from `perftest` in two ways:

- The handshake port is not 18515 but a random port, though it can be specified via the -p parameter
- The data operation is Send, not RDMA Write/Read

We will perform packet capture analysis on `ibv_rc_pingpong`, `ibv_uc_pingpong`, and `ibv_ud_pingpong`, in order to understand the RDMA communication process in depth.

---

## 5.2 Packet Capture Analysis of ibv_rc_pingpong

[pcap capture file](../pcap/pingpong_rc.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/pingpong_rc.pcap 'port 12345 or udp port 4791'

# Server, using -p to specify the TCP port as 12345
ibv_rc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345

# Client
ibv_rc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345 10.1.16.61
```

The test output is as follows:

```bash
expert@k8s-61:~$ ibv_rc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345
  local address:  LID 0x0000, QPN 0x000020, PSN 0x83be68, GID ::ffff:10.1.16.61
  remote address: LID 0x0000, QPN 0x000021, PSN 0x59dfd7, GID ::ffff:10.1.16.62
10240 bytes in 0.00 seconds = 33.02 Mbit/sec
5 iters in 0.00 seconds = 496.20 usec/iter

expert@k8s-62:~$ ibv_rc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345 10.1.16.61
  local address:  LID 0x0000, QPN 0x000021, PSN 0x59dfd7, GID ::ffff:10.1.16.62
  remote address: LID 0x0000, QPN 0x000020, PSN 0x83be68, GID ::ffff:10.1.16.61
10240 bytes in 0.00 seconds = 38.00 Mbit/sec
5 iters in 0.00 seconds = 431.20 usec/iter
```

### The Timing as Seen from the Capture

![pingpong_rc](../assets/pingpong_rc.png)

```bash
Packets 1-3    TCP three-way handshake   connection setup
Packets 4-8    TCP data exchange         handshake: exchange QPN/PSN/GID
Packets 9-11   TCP four-way teardown     connection close
Packets 12+    RRoCE data                RC Send Only + RC Acknowledge appearing alternately
```

After exchanging information such as QPN, PSN, and GID, `ibv_rc_pingpong` immediately closes the TCP connection, and subsequently relies entirely on RDMA communication to complete the data exchange. In stark contrast are the tools from the previous chapter such as `ib_write_bw`, which choose to keep the TCP connection as a test control channel, used to implement state synchronization such as the start and end of the test. Both implementations are entirely reasonable; they are essentially just different engineering choices made by the developers.

Additionally, this capture also shows a very clear pingpong pattern, a strict four-step round of **ping→ack→pong→ack**:

```bash
Packet 12  62 → 61  RC Send Only   1082 bytes   client sends data (ping)
Packet 13  61 → 62  RC Acknowledge   62 bytes   server ACK
Packet 14  61 → 62  RC Send Only   1082 bytes   server sends data back (pong)
Packet 15  62 → 61  RC Acknowledge   62 bytes   client ACK
Packet 16  62 → 61  RC Send Only   1082 bytes   next round's ping
...
```

---

## 5.3 Packet Capture Analysis of ibv_uc_pingpong

[pcap capture file](../pcap/pingpong_uc.pcap)

### Test Environment Setup

```bash
# Capture on the server
sudo tcpdump -i ens160 -w /tmp/pingpong_uc.pcap 'port 12345 or udp port 4791'

# Server, using -p to specify the TCP port as 12345
ibv_uc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345

# Client
ibv_uc_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345 10.1.16.61
```

![pingpong_uc](../assets/pingpong_uc.png)

UC's behavior is basically the same as RC, with only one key difference: **there is no ACK**.

```bash
RC pingpong:
  62 → 61  RC Send Only      data
  61 → 62  RC Acknowledge    ACK   ← hardware replies automatically
  61 → 62  RC Send Only      data
  62 → 61  RC Acknowledge    ACK   ← hardware replies automatically

UC pingpong:
  62 → 61  UC Send Only      data
  61 → 62  UC Send Only      data  ← no ACK, replies with data directly
  62 → 61  UC Send Only      data
  61 → 62  UC Send Only      data
```

---

## 5.4 Packet Capture Analysis of ibv_ud_pingpong

[pcap capture file](../pcap/pingpong_ud.pcap)

### Test Environment Setup

```bash
# Capture on the server:
sudo tcpdump -i ens160 -w /tmp/pingpong_ud.pcap 'port 12345 or udp port 4791'

# Server
ibv_ud_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345

# Client
ibv_ud_pingpong -d rxe0 -g 1 -n 5 -s 1024 -p 12345 10.1.16.61
```

### UD's MTU Limit

UD does not support fragmentation, and a single packet cannot exceed the MTU (1024 bytes in the current test environment), so `-s 1024` is right at the boundary. You can try `-s 1025`, which will error out directly:

```bash
expert@k8s-61:~$ ibv_ud_pingpong -d rxe0 -g 1 -n 5 -s 1025 -p 12345
Requested size larger than port MTU (1024)
```

### Analysis of the DETH Header Structure

![pingpong_ud](../assets/pingpong_ud.png)

The Datagram Extended Transport Header is exclusive to UD; RC/UC do not have it. Taking packet 12 as an example:

```bash
Queue Key:        0x0000000011111111
Source QPN:       0x00000023
```

**Source QPN** is the QPN identifying the source side. When RC/UC establish a connection, the two parties' QPNs are already bound, so when a packet is received it is known where it came from. UD has no connection state, and the receiver's QP may simultaneously receive packets from dozens of different senders; without the Src QPN there would be no way to distinguish them.

**Queue Key** is an access control mechanism. A qkey is set when a UD QP is created, and when a packet is received the hardware compares the Queue Key in the packet with the local QP's qkey, discarding it if they don't match. In the capture it is `0x11111111`, which is the default value of `ibv_ud_pingpong`; both sides use the same value, so it works.

> An analogy: the receiver's QP in UD is a "public inbox," and the Src QPN in the DETH is equivalent to the sender's address on the envelope. The application-layer `ibv_wc` (completion queue entry) returns this Src QPN together with the qkey to the upper layer, and the application itself decides how to handle it.
