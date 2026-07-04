# Afterword: Lossless and Lossy — A Forty-Year Swing of the Pendulum

Looking back now that the book is written, I want to close from the vantage point of deployment reality and engineering economics.

This book spent nineteen chapters trying to do just one thing: start the reader with RDMA's observable behavior and packet-level protocol analysis, build an intuitive feel from there, and then step into the world of InfiniBand to see how a natively lossless fabric — from its link layer to its management plane to routing and QoS — runs as a single coherent whole. As for Ethernet, I spent almost no ink evaluating it, because for a network engineer that "best-effort" mental model is self-evident; the two worldviews meet on their own by the time you reach the end. But once the book is closed, a more fundamental question still hangs in the air: where does each of these technologies actually stand in the real world?

This afterword is my answer to that question. Let me state one thing up front, though: I never make predictions about history. I only try to explain as clearly as I can how the pendulum has swung over the past forty years and what mechanics drove it, and then to sum up.

---

## I. Training and Inference: One Set of RDMA, Two Deployment Realities

Start with what the industry has made public.

On the training side, NVIDIA's InfiniBand remains the default choice for frontier training (a frontier model being a flagship large model whose capabilities sit at the leading edge of the industry): the DGX and SuperPOD reference architectures are built around IB, the XDR generation continues to push SHARP in-network computing, hardware-level adaptive routing, and deterministic tail latency as its core selling points, and NVIDIA's financials show InfiniBand revenue nearly doubling within a single quarter of fiscal 2026.[1]

In the public domain there is almost no scale-matched, methodologically rigorous, independently verifiable head-to-head test of IB versus RoCE. Small-scale lab tests never traverse an intermediate switch, so the congestion point never materializes; finding the two "comparable" is a foregone conclusion, and it proves nothing. So here I only lay out the deployment facts and leave the conclusions to the reader.

The flagship case for the IB camp is the successive training supercomputers Microsoft built for OpenAI: from the systems of the GPT-3 and GPT-4 era, to Eagle (14,400 H100s, NDR InfiniBand), ranked third on the Top500 in 2023, to the world's first large-scale GB300 NVL72 cluster in 2025 (Quantum-X800 InfiniBand + SHARP v4) — IB all the way.[2] Industry figures cited by _The Register_ put roughly ninety percent of AI clusters on IB as of early 2024.[3]

Heavyweight cases in the Ethernet camp have appeared just as steadily. In 2024 Meta built two 24K-GPU H100 clusters as a controlled pair, one RoCE and one IB; after tuning they performed comparably, and the largest Llama 3 model was trained on the RoCE cluster.[4] xAI's Colossus trained Grok on a hundred thousand (later expanded to two hundred thousand) Hopper GPUs over Spectrum-X Ethernet.[3] The latest development, in 2026, is OpenAI's MRC networking scheme: RoCE-based, absorbing UEC technology, and already deployed on several of its own largest GB200 training supercomputers.[5] Even the onetime biggest standard-bearer of the IB camp has gone over to the Ethernet family for its latest-generation fabric.

On the inference side, the scales tip even more visibly. Building an inference cluster on IB costs 1.5 to 2.5 times more per port than an Ethernet solution, and with inference workloads being atomized and lacking any global synchronization semantics, that premium is hard to justify;[6] tensor parallelism, moreover, is pinned inside the single-machine NVLink domain by token latency, so the traffic that genuinely crosses nodes places demands on the network closer to those of a traditional data center. The infrastructure provider Spheron has offered a rule-of-thumb threshold: **above a 25% communication-to-computation ratio it is worth evaluating a dedicated IB cluster; below 10%, stay on Ethernet.**[7] Training and inference happen to fall on opposite sides of that line.

NVIDIA itself is hedging both ends: IB to harvest training, Spectrum-X to harvest inference and the Ethernet camp (annualized revenue already past ten billion dollars) — and IDC data shows that by Q1 2026 NVIDIA had become the largest data-center Ethernet switch vendor.[1] Just like that, a GPU company had climbed to the top of the traditional networking vendors' core product category.

There are also two routes that sidestep the IB-versus-RoCE dispute entirely: AWS uses its home-grown SRD protocol and EFA NICs to underpin Project Rainier, training Claude on over a million Trainium2 chips,[8][9] and AWS doesn't even offer an IB or RoCE option;[10] Google's TPU pods, meanwhile, have always been built on its own ICI interconnect and optical circuit switching, with no intersection with the IB ecosystem at all. In fairness, though, SemiAnalysis's assessment is that EFA still trails IB, Spectrum-X, and mainstream RoCEv2 solutions on raw network performance;[11] these home-grown routes win on total cost of ownership and supply-chain independence, not on performance itself.

With the public record laid out, here is my summary.

**In the scenario of large training clusters, IB is still the optimal solution — but the domain over which "optimal" holds needs to be spelled out.** It is the optimal solution under the objective function of "determinism plus turnkey": if what you want is out-of-the-box predictable tail latency, a reference architecture validated across hundreds of SuperPOD deployments, and the in-network aggregation bonus that SHARP provides, and if training time converts directly into money, then IB has no rival to this day. And if you observe the players who have successfully sidestepped IB, you find they fall into two groups. One group is organizations like Meta, Google, AWS, and OpenAI, and they share one glaring trait: every one of them runs a network engineering team hundreds strong, and some design their own silicon and protocols. The other group is xAI's Colossus, which looks like it chose Ethernet but in fact bought NVIDIA's Spectrum-X turnkey package — switches, SuperNICs, and congestion-control algorithms all from the same source; what it left behind was only the IB product line, not NVIDIA's ecosystem. In other words, the exceptions prove the rule precisely: **there are only two tickets out of IB: either take the systems-engineering problems that IB solves for you at the architectural level and carry them yourself, with your own headcount; or buy a different bundled package from the same vendor.** For the vast majority of organizations without hyperscaler engineering resources, the first ticket is unaffordable; and the very existence of the second ticket is exactly what shows how valuable a commodity "turnkey determinism" is.

Once you reach inference and edge clusters, the economics swing to RoCE — and it isn't close. This is not a judgment about technical superiority but a consequence of the traffic profile: when cross-node communication no longer sits on the critical path of computation, every cent of premium paid for lossless determinism is money wasted.

One set of RDMA semantics, two deployment realities. That in itself is a signal worth pondering: what decides the fate of a networking technology has never been the protocol alone, but the economics of the traffic it serves.

---

## II. Forty Years in Hindsight: Lossless Fabrics Have Always Been Niche — but Niche Doesn't Mean No Market

Pull the lens back across nearly forty years of networking history and a clear lineage comes into view.

Fibre Channel used buffer-to-buffer credit to build a lossless SAN for storage traffic; InfiniBand carried the same credit-based flow-control philosophy into HPC. These two systems share the same ancestry: both belong to the camp where "the network bears responsibility for determinism." In this camp the network is not merely a conduit that forwards data, but a system that makes architectural-level commitments to no packet loss, no reordering, and predictable latency.

This lineage shares a common fate: they all survived, and they all reign in their own niches — FC remains to this day the de facto standard for enterprise mission-critical storage, and IB ruled HPC for over two decades and found a second act in the AI era; but not one of them ever took the position of "general-purpose connectivity." That position has always been held by best-effort Ethernet + IP.

Niche, but not short of a market. Under specific conditions the lossless fabric genuinely does provide the optimal solution, and that must be honestly acknowledged. The question is: what draws the boundary of that "specific condition"?

My answer is: **the radius of complexity.**

A lossless fabric is, in essence, a piece of systems engineering. Its correctness depends on no single device, but on the consistency of global state: credit initialization and reclamation must close the loop on every link, buffer planning must match the topology and traffic matrix, routing tables must be computed and pushed out uniformly by a centralized management plane (IB's SM, FC's Fabric Services), and partitioning and QoS policy must coordinate end to end. Chapters 11 through 17 of this book took these mechanisms apart one by one: none is hard on its own; what's hard is that they must all be correct at the same time. This means the complexity of a lossless network is not a point but a radius: **the more nodes take part in the coordination and the greater the physical distance spanned, the more the cost of maintaining global consistency grows super-linearly.**

This is why the scale of a lossless fabric has a ceiling. IB's single-subnet limit, the SM's centralized bottleneck, FC's assorted constraints on fabric size — at bottom, these are different manifestations of the same constraint. Imagine pushing lossless semantics to the scale of the internet — maintaining credit consistency across autonomous domains, making global routing obey a single management plane — and the radius of complexity simply blows up. The success of the internet lies precisely in doing the exact opposite: compressing the radius of correctness down to single-hop forwarding plus endpoint protocols, so that the network holds no state that must be globally consistent.

So the conclusion forty years of history offers is symmetric: in scenarios where the radius is controllable (it can be a pod, or a machine room, or a SAN spanning an active-active data-center pair), the lossless fabric is a repeatedly validated optimal solution; but once the radius goes out of control, its systems-engineering nature turns from an asset into a liability. The lossless fabric therefore has its own space to live, but the boundary of that space is drawn by the radius of complexity, not by technical superiority.

---

## III. The Rational Endgame: Lossy Fabric Plus Endpoint Reliability

If you push the observations above to their conclusion, you arrive at a judgment: **weighing engineering complexity, cost, and scale together, the combination of "lossy fabric plus endpoint reliability" is the more rational choice.** The basis for this judgment is the historical logic that has recurred over the past forty years.

This conclusion has a forty-year empirical case behind it: Ethernet + TCP. The network does nothing but best-effort forwarding; reliability, ordering, and congestion control are all handled by the endpoints' protocol stack. It claimed the general-purpose connectivity slot and squeezed a host of connection-oriented specialized networks back into their respective niches — not on the strength of any single technical metric, but on the crushing advantage this division of labor holds at scale: the network stays stateless and can therefore scale out without limit, while complexity is pushed to the endpoints and can therefore be continuously amortized by Moore's Law. This principle was later distilled into a general design maxim, the end-to-end argument. In the forty years since, Moore's Law kept making endpoint compute cheaper, and so the pendulum never swung back to the "let the network bear responsibility for determinism" side.

This round of AI networking's evolution is replaying the same story on the same curve, only the form of the endpoint has been upgraded. Last round's "endpoint reliability" was software TCP running on a general-purpose CPU, trading flexibility for performance; this round's endpoint reliability is a hardware transport engine inside the NIC ASIC, with out-of-order reassembly, precise retransmission, and congestion response all done at line rate in silicon. AWS's SRD has proven in production that this path works: the underlying network is allowed to drop, reorder, and multipath, and reliability is fully backstopped by the NIC endpoints within the Nitro system (AWS's home-grown dedicated hardware offload architecture, which moves infrastructure functions such as networking and storage off the host CPU onto separate chips) — supporting frontier-model training at the scale of a million chips. What UEC is doing is, in essence, turning SRD's private answer into an open-standard public answer.

The economics here are worth unpacking. **Put the complexity into the network, and the complexity concentrates inside the fabric, where it must be digested by operations and global coordination — this is an engineering problem, and engineering problems cannot be replicated at scale. Pull the complexity into the endpoints, and the complexity is amortized across every NIC, solvable once by chip design and replicated without limit by mass production — this is a product problem, and product problems are inherently scalable.** The reason lossy fabric plus endpoint reliability is the "rational choice" is not that it is better on every metric, but that it converts an unreplicable engineering problem into a replicable product problem. On the whole, it genuinely does reduce technical complexity — not by eliminating complexity, but by moving it to the place where it can be digested most efficiently.

Ethernet + TCP was the first canonical case of this paradigm. AWS-EFA-style "lossy network plus hardware endpoint reliability" may well be the next.

---

## IV. PFC, ECN, DCQCN: Intermediate Products Worthy of Respect

So where, then, should we place the PFC, ECN, and DCQCN that today's RoCEv2 depends on to survive? This book spends little ink on them — they deserve a monograph of their own to unfold — but given this book's premise — you can only read the port once you've read the original — I owe the reader an account of where they stand.

My view is: **they are intermediate products — the rational choice for a particular historical stage, but not the endpoint.**

First, their reasonableness. RDMA was born of IB's lossless assumption, and its retransmission mechanism is crude to the point of stubbornness: once a packet is lost, every packet received after it — however intact — is discarded, and the sender must resend everything it transmitted from the loss point onward (go-back-N). Lose one packet, forfeit the entire in-flight window; this cost structure means RDMA cannot tolerate loss. And before UEC arrived, the only way was to make a lossy network simulate lossless behavior — PFC for back-pressure to stem the loss, ECN to signal, DCQCN to adjust the rate; the trio got RoCEv2 running in cost-sensitive scenarios, and that was a perfectly rational engineering decision.

But the cost is just as plain to see; the preface called it "parameter-tuning hell." PFC is a coarse-grained version of IB's credit back-pressure: what it pauses is not a single virtual lane but an entire priority, and it can form pause storms and even deadlock. This coarse granularity is not a design mistake; it is the price a lossy gene must pay to simulate losslessness. PFC is an after-the-fact brake: once the pause frame is sent, in-flight data keeps arriving, so the receiver must reserve headroom buffer for every lossless priority, sized to grow linearly with bandwidth and link distance — and on-chip buffer is precisely the most expensive resource, so economically you can usually afford only one or two lossless classes. Credit-based flow control is prior authorization: before a byte leaves the source, the receiver's buffer has already been accounted for, so there is no "in-flight surprise," and per-VL fine granularity is a natural byproduct. One makes buffer state a protocol primitive; the other bolts a brake onto a link that transmits freely — a world apart at the genetic level. More subtly still: **IB's losslessness is native, its consistency guaranteed by the architecture; RoCE's losslessness is assembled, its consistency guaranteed by operations.** Chase a native design with patches, and you often end up more complex than the native design itself.

UEC will remove the very precondition for this patch system to exist. Selective Retransmission in UET (Ultra Ethernet Transport) compresses the recovery cost of a lost packet from "roll back the entire window" down to "one packet," and packet trimming lets a congested switch truncate a packet to its header and forward that, rather than silently dropping it in place.[12] When the endpoint can digest packet loss at line rate, the network no longer needs to promise not to drop. But be clear: **what UEC eliminates is "network-enforced losslessness," not congestion control itself** — UET comes with a brand-new congestion control of its own, only shifted from "switch back-pressure chain" to "negotiation between endpoints."[12] The Credit-Based Flow Control that UEC retains as an option at the link layer is a candid homage to this very mechanism from FC and IB: per-virtual-lane credit authorization, congestion strangled at the source rather than back-pressured after the fact, and far simpler to deploy than the coupled parameter-tuning of today's trio.[13]

A historical detail worth savoring: PFC was never born for RDMA. It was a component the DCB effort built for FCoE — the previous attempt to make Ethernet simulate losslessness and carry Fibre Channel semantics. FCoE never went far, but RoCE picked the component up and carried the same proposition into the AI era. That is its place in history: inglorious but necessary — holding up a transitional era, then giving way to something designed for that era from the start. The right attitude toward them is respect, not attachment.

---

## V. Ethernet's Genes Haven't Changed — the Layers Above Are Covering for Them

Finally, a word on "Ethernet's path-selection capability."

Traditional Ethernet's Layer 2 is "dumb": broadcast, flooding, spanning tree, and next to no path-selection capability. That dumbness is by design — it is the price of simplicity, and also its source. The real routing intelligence lives at Layer 3, while inside the subnet packets take their chances; this division of labor worked well in an era dominated by north-south traffic.

The AI fabric broke that balance. The traffic matrix of collective communication is sparse and enormous, and a handful of elephant flows can fill a link; ECMP's flow-based hashing polarizes frequently under such conditions, with several large flows hashed onto the same path while the remaining paths sit idle. Chapter 14 of this book covered IB's answer to this: adaptive routing, where the switch continuously monitors real-time link load and queue depth and dynamically steers around congestion. This is a closed-loop mechanism. In the traditional Ethernet system, no corresponding mechanism exists inside the subnet.

Packet spraying is the Ethernet world's answer, but it's worth being precise about how far it actually goes. UET has the sender vary the entropy value per packet (that is, the input field to the ECMP hash — in practice usually rewriting the UDP source port number per packet), scattering packets evenly across all equal-cost paths, with the receiver responsible for reassembly.[14] This is indeed a step ahead of traditional ECMP: no longer does one large flow lock down one path, and static hotspots disappear.

But this is still not IB's adaptive routing, and the difference is not one of degree but of kind: adaptive routing is closed-loop — the switch makes decisions from real-time load, avoiding whichever path is congested; packet spraying is open-loop — the sender only spreads evenly, with no awareness whatsoever of whether any given path is congested at this moment. Even distribution is not load-aware distribution: it fixes the hotspots created by ECMP's static hashing, but it cannot supply dynamic avoidance under real-time congestion.

More noteworthy still is where the compensation happens: this capability is implemented on the sender's NIC, not in the Layer 2 switch itself. Ethernet's Layer 2 switching gear has not become smarter — it is still the same broadcast, flood, and forward gene; the one actually doing the work is the endpoint, tweaking a field in the packet header. Traditional Layer 3 path selection alone can no longer meet AI traffic's demands on path utilization inside the fabric, but Ethernet, facing this gap, has not performed surgery on Layer 2's simple, clumsy genes — instead it wraps a compensating layer around both ends. What fills the gap is endpoint-side engineering, not Layer 2 itself acquiring the kind of native, closed-loop path-selection intelligence that IB has.

---

## Closing: The Spiral

Having written this far, I can gather the book's threads into a single picture.

The history of networking technology swings on a pendulum: at one end, centralized deterministic guarantees represented by FC and IB, where the network is responsible for everything; at the other, distributed best-effort plus endpoint intelligence represented by Ethernet + TCP, where the network only forwards. The direction of each swing is decided by the hardware cost curve of the day: when endpoint compute is expensive, building complexity into the network is rational; when endpoint compute is cheap, pulling complexity back to the endpoints is rational. Over the past forty years, Moore's Law kept pushing the cost curve toward the endpoint side, so every backswing of the pendulum ultimately landed on the "lossy fabric plus endpoint reliability" side.

But this is not a simple loop; it is a spiral. In the round where Ethernet + TCP won general-purpose connectivity, endpoint reliability rested on a software protocol stack running on the CPU; in this round, where UEC challenges IB and RoCEv2, it rests on a hardware transport engine inside the NIC ASIC. The same end-to-end argument, re-validated at a performance level an order of magnitude higher. Each "return" departs again from a higher turn of the spiral.

The lossless fabric has never disappeared. As long as there remain in the world scenarios willing to pay a premium for determinism — HPC's tightly coupled simulations, finance's microsecond arms race, a frontier lab's obsession with every last point of MFU (Model FLOPs Utilization), enterprise mission-critical storage's near-obsessive pursuit of zero-loss, zero-jitter stability (FC remains to this day the de facto standard for banks' core transaction systems and large-database storage) — the lossless fabric will have a place to live. This is a logic validated over and over across the past forty years.

Proceeding from the same historical logic, my judgment is: viewed through the three variables of engineering complexity, cost, and scale, lossy fabric plus endpoint reliability is the more rational choice in most scenarios. So why spend an entire book on InfiniBand?

Because IB carried the "network bears the determinism" school of engineering to textbook completeness: from credit-based flow control to the centralized SM, from VL arbitration to in-network computing, it is the purest and most self-consistent specimen of that direction. Let me borrow an analogy from the roots of Chinese culture: read only the classics of the Confucian school, and what you get is probably not so different in essence from the Bible or the Buddhist sutras — one more teaching that urges people toward the good. Only once you have also understood Legalism, Daoism, and Mohism — Confucianism's opposites and foils — do you know what Confucianism was answering, what it rejected, and what it quietly borrowed. The Ethernet storyline works the same way: unless you first see what it looks like when IB takes "the network bears the determinism" to its extreme, you cannot read what RoCEv2 is dodging, what UEC is stealing from it, and what SRD is pushing back against. As the classical idiom has it, stones from other hills can polish your own jade — and IB is the most complete such stone. Specific technologies will keep going obsolete, but the structure of this pendulum's forty-year swing is a pattern you can read straight out of history.

That is what this book truly means to deliver.

Thank you for reading this far.

---

## References

Below are the public sources for the data and conclusions cited in this afterword, arranged in order of first appearance in the text, for further reading and verification. All links accessed in July 2026.

1. **Scale of NVIDIA's networking business**: near-ten-billion-dollar quarterly networking revenue, up more than twofold year over year; Spectrum-X annualized revenue past ten billion; IDC data showing NVIDIA became the largest data-center Ethernet switch vendor in Q1 2026
   AInvest, _The $6.9 Billion Acquisition That Turned Nvidia Into a Networking Giant_ — <https://www.ainvest.com/news/6-9-billion-acquisition-turned-nvidia-networking-giant-2606/>
   SDxCentral, _Nvidia: Networking is booming but your networks cost nothing_ — <https://www.sdxcentral.com/news/nvidia-networking-is-booming-but-your-networks-cost-nothing/>
   Windows News, _NVIDIA Becomes Largest Data Center Ethernet Switch Vendor in Q1 2026, IDC Says_ — <https://windowsnews.ai/article/nvidia-becomes-largest-data-center-ethernet-switch-vendor-in-q1-2026-idc-says.431907>

2. **The IB training supercomputers Microsoft built for OpenAI**: Eagle (14,400 H100s, NDR InfiniBand, third on the 2023 Top500); the world's first large-scale GB300 NVL72 cluster (Quantum-X800 InfiniBand + SHARP v4)
   Microsoft Azure Blog, _NVIDIA GB300 NVL72: Next-generation AI infrastructure at scale_ — <https://azure.microsoft.com/en-us/blog/microsoft-azure-delivers-the-first-large-scale-cluster-with-nvidia-gb300-nvl72-for-openai-workloads/>
   Microsoft Tech Community, _Annual Roundup on AI Infrastructure Breakthroughs for 2023_ — <https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/annual-roundup-of-ai-infrastructure-breakthroughs-for-2023/4097737>

3. **xAI Colossus**: a hundred thousand (later expanded to two hundred thousand) Hopper GPUs, Spectrum-X Ethernet + BlueField-3 SuperNIC; the industry figure that "roughly ninety percent of AI clusters used IB as of early 2024"
   NVIDIA Newsroom, _NVIDIA Ethernet Networking Accelerates World's Largest AI Supercomputer, Built by xAI_ — <https://nvidianews.nvidia.com/news/spectrum-x-ethernet-networking-xai-colossus>
   The Register, _xAI's 100,000 H100 Colossus is glued together using Ethernet_ — <https://www.theregister.com/2024/10/29/xai_colossus_networking/>

4. **Meta's paired 24,576-GPU clusters**: one RoCE (Arista 7800), one Quantum-2 InfiniBand; comparable performance after tuning, with the largest Llama 3 model trained on the RoCE cluster
   Meta Engineering, _Building Meta's GenAI Infrastructure_ — <https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/>
   Meta Engineering, _How Meta trains large language models at scale_ — <https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/>

5. **OpenAI MRC**: a training network solution based on RoCE, absorbing UEC technology and layering on SRv6 source routing, already deployed on the GB200 training supercomputers at OCI Abilene and Microsoft Fairwater
   OpenAI, _Supercomputer networking to accelerate large scale AI training_ — <https://openai.com/index/mrc-supercomputer-networking/>

6. **RoCE's economic advantage for inference clusters**: IB's 1.5–2.5× higher per-port cost, the atomized nature of inference workloads, RoCEv2's Layer 3 routability and its fit for edge deployment
   NADDOD, _Training vs Inference: Why Your AI Network Architecture Needs to Be Different_ — <https://naddod.medium.com/training-vs-inference-why-your-ai-network-architecture-needs-to-be-different-97a5718f14aa>

7. **The communication-to-computation ratio decision threshold**: evaluate IB above 25%, stay on Ethernet below 10%; a comparison of fabric cost amortization for IB versus RoCEv2
   Spheron, _GPU Networking for AI Clusters: InfiniBand vs RoCE vs Spectrum-X Decision Guide (2026)_ — <https://www.spheron.network/blog/gpu-networking-infiniband-roce-spectrum-x-guide/>

8. **Project Rainier**: nearly half a million (later over a million) Trainium2 chips, third-generation EFA petabit-class interconnect, a two-tier NeuronLink (intra-domain) + EFA (inter-domain / cross-data-center) architecture
   Data Center Dynamics, _AWS activates Project Rainier cluster of nearly 500,000 Trainium2 chips_ — <https://www.datacenterdynamics.com/en/news/aws-activates-project-rainier-cluster-of-nearly-500000-trainium2-chips/>
   Amazon, _AWS's Project Rainier: the world's most powerful computer for training AI_ — <https://www.aboutamazon.com/news/aws/aws-project-rainier-ai-trainium-chips-compute-cluster>

9. **Anthropic's hardware strategy**: over a million Trainium2 chips to train and run Claude; a diversified hardware strategy using Trainium / TPU / NVIDIA GPU simultaneously
   Anthropic, _Anthropic and Amazon expand collaboration for up to 5 gigawatts of new compute_ — <https://www.anthropic.com/news/anthropic-amazon-compute>
   Anthropic, _Anthropic expands partnership with Google and Broadcom for multiple gigawatts of next-generation compute_ — <https://www.anthropic.com/news/google-broadcom-partnership-compute>

10. **EFA / SRD**: the only RDMA fabric on AWS (no IB or RoCE offered); OS-bypass architecture, NIC-side reliable transport, 3.2 Tbps EFAv3 spec on Trn2 instances
    NVIDIA Dynamo Documentation, _EFA (RDMA over AWS Fabric) on EKS_ — <https://docs.nvidia.com/dynamo/kubernetes-deployment/cloud-provider-guides/aws/efa-rdma-over-aws-fabric>
    AWS, _Elastic Fabric Adapter (EFA)_ official documentation — <https://aws.amazon.com/hpc/efa/>

11. **SemiAnalysis independent assessment**: EFA performance still trails InfiniBand, Spectrum-X, and mainstream RoCEv2 solutions; analysis of Trainium2's memory-bandwidth / TCO advantages
    SemiAnalysis, _Amazon's AI Resurgence: AWS & Anthropic's Multi-Gigawatt Trainium Expansion_ — <https://newsletter.semianalysis.com/p/amazons-ai-resurgence-aws-anthropics-multi-gigawatt-trainium-expansion>

12. **UET mechanisms**: selective retransmission, packet trimming, dual congestion-control mechanisms (sender-side windowed and receiver-side credit)
    Ultra Ethernet Consortium, _Ultra Ethernet Specification v1.0_ — <https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UE-Specification-6.11.25.pdf>
    Ultra Ethernet Consortium, _Ultra Ethernet Specification Update_ — <https://ultraethernet.org/ultra-ethernet-specification-update/>

13. **UE link-layer optional extensions**: mechanism details of Credit-Based Flow Control, Link Layer Retry, and Packet Trimming
    Tom's Hardware, _Ultra Ethernet: The data-center interconnection of tomorrow detailed_ — <https://www.tomshardware.com/networking/ultra-ethernet-the-data-center-interconnection-of-tomorrow-detailed>
    Arista, _Demystifying Ultra Ethernet_ — <https://blogs.arista.com/blog/demystifying-ultra-ethernet>

14. **UE architectural design principles**: per-packet multipath (packet spraying), lossy operation to sidestep head-of-line blocking, endpoint-side out-of-order reassembly
    Hoefler et al., _Ultra Ethernet's Design Principles and Architectural Innovations_ (arXiv preprint) — <https://arxiv.org/pdf/2508.08906>
    HPCwire, _Ultra Ethernet Has Arrived: One Network to Rule Them All?_ — <https://www.hpcwire.com/2025/09/09/ultra-ethernet-has-arrived-one-network-to-rule-them-all/>

15. **The end-to-end argument**
    J. H. Saltzer, D. P. Reed, D. D. Clark, _End-to-End Arguments in System Design_, ACM Transactions on Computer Systems, Vol. 2, No. 4, 1984.

> Note: The specific figures in third-party analyses and media reports (revenue, cost multiples, and so on) are largely statistics or estimates by each organization based on public information, and may differ across methodologies; the text has tried to use ranges or orders of magnitude where possible. Vendor financial data is authoritative per official filings (SEC filings).
