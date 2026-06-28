# Chapter 2: RDMA Connection Management

The previous chapter mentioned that RDMA RC needs to establish a connection before transferring data. Taking the Write operation as an example, the sender's NIC needs to know the peer's QPN, target memory address, rkey, and other key information before it can construct a valid packet and send it out. This information must be negotiated by both parties in advance, before communication begins. This connection-establishment phase is called bootstrap.

For the bootstrap phase, the industry generally has two solutions:

**Out-of-band communication**: use a separate transport channel (such as Ethernet + TCP/IP) to exchange this information first, then begin RDMA communication. The most common approach is to use a TCP Socket: the two ends first establish a TCP connection, send each other their QPN, memory address, rkey, and so on, and after the handshake completes they switch to RDMA transport.

**In-band communication**: use the RDMA network itself to provide connection establishment and address exchange services, without relying on an additional transport channel. This mechanism is called RDMA CM (Connection Manager).

---

## 2.1 What Information Needs to Be Exchanged

Taking RC as an example, the two parties need to exchange at least the following information before the QP can enter a state where it can send and receive data:

- **QPN (Queue Pair Number)**: the peer QP's number, equivalent to a "port number" in the RDMA world, telling the NIC which QP on the other side the data should be handed to.

- **PSN (Packet Sequence Number)**: the initial packet sequence number, used by the receiving end to verify packet order.

- **Virtual memory address and rkey**: these two are only needed for one-sided operations (e.g., Write, Read). To operate the peer's memory directly, the sender must know the target memory's virtual address and the corresponding rkey (the remote access key). For two-sided operations (Send), the receiver itself decides which buffer the data lands in, so the sender doesn't need to know the peer's memory address, and these two items are therefore not exchanged.

- **GID/LID (IB addressing information)**: here's something counterintuitive. Since the two parties are exchanging communication information, they must already know the peer's address, so why exchange IB addressing information here? This is really a matter of engineering practice. Application software (e.g., AI training frameworks) tends to live in the world of IP, but the GPU/CPU needs to access the memory of a remote machine over IB, which requires knowing the peer's addressing information within IB.

---

## 2.2 The Out-of-Band Approach: Exchanging Information over TCP

The out-of-band approach is simple and direct: before RDMA communication, first establish an ordinary connection over TCP and exchange the connection information mentioned earlier. After both parties obtain this information, each completes the configuration of its QP state, and then RDMA data transfer begins.

Surveying the mainstream RDMA testing software and applications in the industry today, after using TCP to establish the QP, there remain some implementation differences in engineering practice, depending on the choices made by the application developer or architect:

- **Whether the TCP connection needs to be kept alive is not mandated at the protocol level.** Once the information exchange is complete, the QP already maintains the connection state, so in principle this TCP connection can be closed immediately. But many implementations choose to keep it as a heartbeat channel or a means of fault detection. Once the TCP connection drops, one can infer that the peer may have failed, thereby triggering reconnection or error handling. Whether to keep it or close it is essentially a trade-off between reliability and resource consumption, with no single correct answer.

- **The underlying network type affects the deployment architecture of the out-of-band approach.** In an InfiniBand network, the IB Fabric itself typically does not carry TCP/IP traffic, so out-of-band communication must rely on a separate Ethernet network. In a RoCEv2 network, however, RDMA itself runs over Ethernet, so the out-of-band TCP connection can directly reuse the existing links without additional hardware. Therefore, the IB solution usually requires building a separate network for the "control plane," whereas the RoCEv2 solution can let the control plane and data plane share the same Ethernet network.

---

## 2.3 The In-Band Approach: RDMA CM

The problem with the out-of-band approach is that it throws all the responsibility for connection management onto the application: you have to write the TCP handshake logic yourself, define the address-exchange format yourself, and decide the connection lifecycle yourself. For systems that need to manage a large number of connections, this code is not trivial.

RDMA CM (Connection Manager) is a connection management mechanism provided by the Linux kernel. It encapsulates the handshake process described above, completing address resolution and connection negotiation through the RDMA network itself, so the application no longer needs to maintain an extra TCP channel.

The most practical thing about RDMA CM is that it provides a programming interface that corresponds closely to the Socket API. In other words, **RDMA CM can be seen as a drop-in replacement for the out-of-band TCP handshake code**, and developers familiar with TCP programming can apply the Socket programming model almost directly. The correspondence between the two sets of APIs is roughly as follows:

| Socket API  | RDMA CM API         | Purpose                       |
| ----------- | ------------------- | ----------------------------- |
| `socket()`  | `rdma_create_id()`  | Create a communication endpoint |
| `bind()`    | `rdma_bind_addr()`  | Bind the local address        |
| `listen()`  | `rdma_listen()`     | Listen for connection requests |
| `connect()` | `rdma_connect()`    | Initiate a connection         |
| `accept()`  | `rdma_accept()`     | Accept a connection           |
| `close()`   | `rdma_destroy_id()` | Destroy the endpoint          |

During `rdma_connect()` and `rdma_accept()`, RDMA CM automatically completes the exchange of connection information such as QPN and PSN, as well as the state transitions of the QP, so developers don't need to handle these low-level details manually. After the connection is established, the data transfer phase still uses the standard Verbs interface; RDMA CM is responsible only for the connection-establishment process and does not intervene in the data path.

More importantly, RDMA CM establishes connections using the in-band approach. For a pure InfiniBand network, this means the connection-establishment process requires no additional Ethernet deployment and can be completed relying solely on the IB network itself.

---

## 2.4 IB Addressing Information

Earlier we mentioned that the two communicating parties need to exchange some addressing information when establishing a connection. Because of differences in the underlying network type (IB/RoCEv2) and the connection-establishment method (in-band/out-of-band), the actual handling differs as well.

The RoCEv2 scenario is relatively simple. Since RoCEv2 runs over Ethernet, the GID (Global Identifier) is derived directly from the NIC's IP address; the two are in fact two representations of the same address at different protocol layers.

- In the out-of-band TCP approach, the two parties pack their respective GIDs into the handshake message and exchange them with each other;
- In the RDMA CM approach, the application only needs to provide the peer's IP address, and CM internally completes the GID resolution automatically, with the application layer being entirely unaware of the existence of the GID.

> Note: In RoCEv2, the GID plays a dual role: to the RDMA layer it is the endpoint identifier; to the RNIC hardware it is also an important source of information for constructing the outer IP header.

The pure IB scenario is relatively more complex, because IB has its own independent addressing system. In it, the LID (Local IDentifier) is dynamically assigned by the SM, and there is no natural mapping between the GID and the IP address.

- In the out-of-band TCP approach, the two parties must explicitly exchange the LID or GID over the TCP channel.
- In the RDMA CM approach, this work is done internally by CM. The traditional method uses IPoIB to map the IP address to a GID, then queries the SA Path Record based on the GID; modern implementations can also use the native IB address (GID) directly, skipping the address mapping and querying the SA Path Record directly. After completing address and path resolution, CM completes the subsequent handshake process within the IB network, and the entire process is completely transparent to the application.

> Note: There's no need to dwell too much on the details of GID and IB addressing here; Chapter 8, "InfiniBand Fundamentals," will give a detailed introduction to IB's addressing system and address resolution process.

## 2.5 Practical Use Cases

The two approaches solve the same problem: safely delivering parameters such as QPN, PSN, memory address, and rkey to the peer before RDMA data transfer begins.

In practice, the choices made by different software stacks vary widely:

In the IB scenario, both approaches coexist, and the dividing line lies in the positioning of the software stack.

- Production IB clusters almost always have a management Ethernet network, which the out-of-band approach can directly reuse. This is exactly what NCCL (NVIDIA Collective Communications Library) does: the bootstrap phase uses an out-of-band TCP socket to exchange QPN and rkey, the RDMA data transfer phase goes over IB, and the entire process does not depend on RDMA CM.

> There is a fundamental reason why NCCL chooses TCP bootstrap: the application framework lives entirely in the world of IP addresses and does not need to be aware of IB's addressing system (LID, GID). The IB network is treated as a pure high-speed channel, and the discovery of ranks (independent computational execution units) and connection management are all handled at the IP layer. Moreover, the bootstrap code can be fully reused across IB and RoCEv2, and the topology management and fault tolerance logic for multi-QP, multi-rail setups are also controlled by the framework itself.

- In the HPC field, MPI/UCX takes a different route, using IB's own UD (Unreliable Datagram) for wireup, completing the establishment of RC QP connections within the IB network, truly achieving independence from any out-of-band channel.
- Storage protocol stacks (NVMe-oF, iSER) lean more toward RDMA CM, valuing its standardized interface and operational manageability.

In the RoCEv2 scenario, the logic of AI training frameworks is exactly the same as in the IB scenario.

- NCCL and PyTorch distributed likewise use TCP sockets or MPI collective operations to complete the bootstrap.
- Storage protocol stacks likewise favor RDMA CM. With the underlying layer switched to RoCEv2, the selection logic of the software stack does not change at all.

Behind this lies a dividing line that runs throughout: application frameworks choose out-of-band, protocol stacks choose in-band.

- Frameworks need to control addressing, topology, and fault tolerance themselves, and the out-of-band approach gives them complete control;
- Protocol stacks need standardized interfaces and interoperability, and RDMA CM's Socket-like API satisfies precisely this need.

No matter which approach is used, after the handshake ends the data path is exactly the same: standard Verbs operations and identically structured IB packets. Later chapters will use packet captures to show you how these parameters exchanged during the bootstrap phase actually appear in real packets.

## 2.6 The Relationship Between NCCL and RDMA

A term frequently mentioned in the AI field is NCCL (NVIDIA Collective Communications Library). What exactly is its relationship with RDMA?

NCCL is a collective communication library released by NVIDIA, designed specifically for multi-GPU distributed training scenarios. In large-model training, hundreds or even thousands of GPUs need to synchronize gradients after every iteration, a process that involves complex cross-node data exchange. NCCL encapsulates this complexity and provides upper-layer AI frameworks (PyTorch, JAX, etc.) with collective communication primitives such as AllReduce, AllGather, and Broadcast. Developers only need to call a single `ncclAllReduce()`, and the cross-node gradient synchronization is completed automatically, with no need to worry about any networking details.

The relationship between NCCL and RDMA can be likened to that between **the C language and assembly language**.

- RDMA is the "assembly language": precise in operation but low-level, requiring the user to personally manage QP, MR, WR, and CQ, and to explicitly tell the hardware to "write the data to a certain memory address on the peer";
- NCCL is the "C language": it builds a higher-level abstraction on top of RDMA semantics, encapsulating all the details of QP establishment, memory registration, ring-algorithm sharding, multi-step transfers, and so on, completely transparently to the upper-layer framework. Just as a C programmer doesn't need to care `how many assembly instructions c = a + b compiles into`, an AI framework engineer doesn't need to care `how many RDMA WRITEs are actually issued behind a single AllReduce`.

NCCL frees AI application developers from networking complexity. But from a network engineer's perspective, NCCL looks more like a "black box." Because every collective communication operation in NCCL is ultimately decomposed at the lower layer into a series of RDMA actions, which are then executed one by one over the IB or RoCE network. So for a network engineer, understanding AI Networking still has to start with RDMA.
