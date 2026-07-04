# 后记：无损与有损——一场四十年的钟摆运动

写完全书再回望，我想从部署现实与工程经济学的角度来收尾。

这本书用了十九章，只想完成一件事：先让读者从 RDMA 的行为和抓包协议分析入手，建立起直观的手感，再走进 InfiniBand 的世界，看一张原生无损的 fabric 从链路层、管理平面到路由与 QoS 是如何作为一个整体运转的。至于以太网，我几乎没有花笔墨评价，因为对网络工程师来说，那套"尽力而为"的心智模型不言自明，两种世界观读到最后自会相遇。但合上书之后，一个更根本的问题依然悬在那里：这些技术在真实世界里各自站在什么位置？

后记就是我对这个问题的回答。不过，我要事先声明一点：我从不对历史做任何预测，我只是把过去四十年这架钟摆是怎么摆的、背后的力学是什么尽量讲清楚，并给出总结。

---

## 一、训练与推理：同一套 RDMA，两种部署现实

先看业界公开的信息。

训练这一侧，NVIDIA 的 InfiniBand 依然是 frontier 训练（frontier model，指能力处于业界最前沿的旗舰大模型）的默认选项：DGX 与 SuperPOD 参考架构以 IB 为中心，XDR 一代继续把 SHARP 在网计算、硬件级自适应路由和确定性尾延迟作为核心卖点，NVIDIA 财报显示 InfiniBand 收入在 2026 财年单季内接近翻倍。[1]

公开领域几乎不存在规模对等、方法严谨、数据可复核的 IB 与 RoCE 对比测试，小规模实验室测试因为跨不过中间交换机、拥塞点根本不存在，测出"两者相当"几乎是必然，说明不了问题。所以这里只陈列部署事实，结论留给读者判断。

IB 阵营的旗舰案例是微软为 OpenAI 打造的历代训练超算：从 GPT-3、GPT-4 时代的系统，到 2023 年 Top500 第三名的 Eagle（14400 颗 H100，NDR InfiniBand），再到 2025 年全球首个大规模 GB300 NVL72 集群（Quantum-X800 InfiniBand + SHARP v4），一路都是 IB。[2] 《The Register》援引的行业口径称，2024 年初约九成 AI 集群用的是 IB。[3]

以太网阵营的重量级案例同样接连出现。Meta 在 2024 年同时建设两个 24K 卡 H100 对照集群，一个 RoCE、一个 IB，调优后性能相当，最大的 Llama 3 模型训练在 RoCE 集群上完成。[4] xAI 的 Colossus 用十万颗（后扩至二十万）Hopper GPU 训练 Grok，走的是 Spectrum-X 以太网。[3] 2026 年更新的动向是 OpenAI 的 MRC 网络方案：基于 RoCE、吸收 UEC 技术，已部署于自己最大的几套 GB200 训练超算。[5] 连 IB 阵营昔日最大的旗手，最新一代 fabric 也转向了以太网系。

推理这一侧，天平倾斜得更明显。用 IB 建推理集群，每端口成本比以太网方案高 1.5 到 2.5 倍，而推理负载原子化、无全局同步语义，这个溢价难以证明合理；[6] 张量并行又被 token 延迟摁在单机 NVLink 域内，真正跨节点的流量对网络的诉求更接近传统数据中心。基础设施服务商 Spheron 给出过一个经验阈值：**通信计算比高于 25% 时值得评估专用 IB 集群，低于 10% 时留在以太网上**。[7] 训练与推理，恰好分布在这条线的两侧。

NVIDIA 自己也在两头下注：IB 收割训练，Spectrum-X 收割推理与以太网阵营（年化收入已破百亿），IDC 数据显示 2026 年 Q1 NVIDIA 已成为最大数据中心以太网交换机厂商。[1] 一家 GPU 公司，就这样在传统网络厂商的核心品类上登顶。

还有两条完全绕开 IB 与 RoCE 之争的路线：AWS 用自研 SRD 协议和 EFA 网卡撑起 Project Rainier，超百万颗 Trainium2 训练 Claude，[8][9] AWS 上甚至不提供 IB 或 RoCE 选项；[10] Google 的 TPU pod 则一直构建在自己的 ICI 互联和光路交换上，与 IB 生态从无交集。但公道地说，SemiAnalysis 的评估认为 EFA 在纯网络性能上仍落后于 IB、Spectrum-X 及主流 RoCEv2 方案，[11] 这些自研路线赢在总体拥有成本和供应链自主，而不是性能本身。

罗列完公开信息，说说我的总结。

**在大型训练集群这个场景里，IB 依旧是最优解，但需要写清楚这个"最优"的定义域。** 它是"确定性 + 交钥匙"这个目标函数下的最优解：如果你要的是开箱即用的可预测尾延迟、经过数百套 SuperPOD 验证的参考架构、SHARP 带来的在网聚合加成，并且训练时长直接等价于金钱，那么 IB 至今没有对手。而观察那些成功绕开 IB 的玩家，会发现他们分成两类。一类是 Meta、Google、AWS、OpenAI 这样的组织，共同点很刺眼：全都拥有数百人规模的网络工程团队，甚至自研芯片与协议。另一类是 xAI 的 Colossus，看似选了以太网，实则买的是 NVIDIA 的 Spectrum-X 交钥匙方案，从交换机、SuperNIC 到拥塞控制算法全套出自同一家，它离开的只是 IB 这条产品线，并没有离开 NVIDIA 的生态。换句话说，例外恰好证明了规则：**绕开 IB 的门票只有两张，要么把 IB 在架构层面替你解决掉的那些系统工程问题，重新用自己的人力扛回来；要么向同一个卖家购买另一套打包方案。** 对绝大多数没有 hyperscaler 工程资源的组织而言，第一张门票买不起；而第二张门票的存在本身，恰恰说明"交钥匙的确定性"这件商品有多值钱。

而到了推理集群和边缘集群，RoCE 的经济优势则毫无悬念地凸显出来。这不是技术优劣的判断，而是流量画像的必然：当跨节点通信不再处于计算的关键路径上，为无损确定性支付的每一分溢价都是浪费。

同一套 RDMA 语义，两种部署现实。这本身就是一个耐人寻味的信号：决定网络技术命运的，从来不只是协议本身，而是它所服务的流量的经济学。

---

## 二、四十年回望：无损 fabric 从来是小众，但小众不等于没有市场

把镜头拉远到近四十年的网络技术史，会看到一个清晰的谱系。

Fibre Channel 用 buffer-to-buffer credit 为存储流量构建了无损的 SAN；InfiniBand 把同样的 credit 流控哲学带进了 HPC。这两套体系同宗同源：它们都属于"网络承担确定性责任"的阵营。在这个阵营里，网络不只是转发数据的通道，而是对不丢包、不乱序、延迟可预测做出架构级承诺的系统。

这个谱系有一个共同的命运：它们都活了下来，而且都在各自的细分领域里称王，FC 至今仍是企业关键业务存储的事实标准，IB 统治了 HPC 二十余年并在 AI 时代迎来第二春；但它们没有一个拿到"通用连接"这个位置，那个位置始终被尽力而为的以太网 + IP 占据着。

小众，但不缺市场。在特定条件下，无损 fabric 确实提供了最优解，这一点必须诚实地承认。问题在于，这个"特定条件"的边界是由什么划定的？

我的答案是：**复杂半径。**

无损 fabric 的本质是一个系统工程。它的正确性不取决于任何单一设备，而取决于全局状态的一致：credit 的初始化与回收要在每一条链路上闭环，缓冲区的规划要与拓扑和流量矩阵匹配，路由表要由集中式的管理平面（IB 的 SM、FC 的 Fabric Services）统一计算下发，分区与 QoS 策略要端到端协同。本书第十一章到第十七章逐一拆解过这些机制：单看每一个都不难，难的是它们必须同时正确。这意味着无损网络的复杂度不是一个点，而是一个半径：**参与协同的节点越多、跨越的物理距离越远，维持全局一致所需的成本就超线性增长。**

这就是为什么无损 fabric 的规模存在天花板。IB 的单子网规模、SM 的集中式瓶颈、FC 对 fabric 尺寸的种种约束，本质上是同一个约束的不同表现。设想把无损语义推到 internet 的尺度，跨越自治域维持 credit 一致性、让全球路由服从一个管理平面，复杂半径会直接爆掉。而 internet 的成功，恰恰在于它做了完全相反的事：把正确性的半径压缩到单跳转发加端点协议，网络不持有任何必须全局一致的状态。

所以四十年的历史给出的结论是对称的：在半径可控的场景里（可以是一个 pod，或者一个机房，或者一张跨越双活数据中心的 SAN 网络），无损 fabric 是经过反复验证的最优解；而一旦半径失控，它的系统工程属性就会从资产变成负债。因此，无损 fabric 有自己的生存空间，但这个空间的边界，是由复杂半径而非技术优劣划定的。

---

## 三、理性的终局：有损 fabric 加端侧可靠

如果把上面的观察推到底，会得到一个判断：**从工程复杂度、成本和规模三个维度综合权衡，"有损 fabric + 端侧可靠"这个组合，是更理性的选择。** 这个判断的依据，是过去四十年反复出现的历史逻辑。

这个结论有一个运行了四十年的实证案例：以太网 + TCP。网络只管尽力转发，可靠性、有序性、拥塞控制全部由端点的协议栈负责。它拿下通用连接的位置、把一众面向连接的专用网络挤回各自的细分领域，靠的不是任何单项技术指标，而是这个分工方式在规模化时的碾压性优势：网络保持无状态因而可以无限水平扩展，复杂性被推到端点因而可以被摩尔定律持续摊薄。这个道理后来被总结成一个通用的设计准则，叫端到端原则。自那以后的四十年里，摩尔定律不断把端点算力做得更便宜，天平也就一直没有摆回"让网络承担确定性责任"那一侧。

这一轮 AI 网络的演进，正在同一条曲线上重演这个故事，只是端点的形态升级了。上一轮的"端侧可靠"是跑在通用 CPU 上的软件 TCP，用灵活性换性能；这一轮的端侧可靠，是网卡 ASIC 里的硬件传输引擎，乱序重组、精确重传、拥塞响应全部以线速在硅片里完成。AWS 的 SRD 用生产环境证明了这条路走得通：底层网络允许丢包、乱序、多路径，可靠性完全由 Nitro（AWS 自研的专用硬件卸载体系，把网络、存储等基础设施功能从主机 CPU 移到独立芯片上处理）体系里的网卡端点兜底，撑起的是百万芯片级的前沿模型训练。UEC 做的事情，本质上是把 SRD 这个私有答案变成开放标准的公共答案。

这里面的经济学值得拿出来分析。**把复杂性放进网络，复杂性就集中在 fabric 里，必须靠运维和全局协同去消化，这是一个工程问题，工程问题无法规模化复制；把复杂性收进端侧，复杂性就被摊薄到每一张网卡上，可以由芯片设计一次性解决、由量产无限复制，这是一个产品问题，产品问题天然可以规模化。** 有损 fabric 加端侧可靠之所以是"理性选择"，不是因为它在每一项指标上更好，而是因为它把一个不可复制的工程问题，转换成了一个可以复制的产品问题。整体上，这确实实实在在地减小了技术复杂度：不是消灭了复杂性，而是把复杂性搬到了它能被最高效消化的位置。

以太网 + TCP 是这个范式的第一个典型案例。类 AWS EFA 的"有损网络 + 硬件端侧可靠"，也许就是下一个。

---

## 四、PFC、ECN、DCQCN：值得尊敬的中间产物

那么，今天 RoCEv2 赖以生存的 PFC、ECN、DCQCN，又该如何安放？本书对它们着墨不多，值得一本专著来展开，但站在全书"看懂原版才能看懂移植"的立场上，对它们的定位必须给出一个交代。

我的看法是：**它们是中间产物：特定历史阶段的理性选择，但不是终点。**

先说合理性。RDMA 诞生于 IB 的无损假设，重传机制简陋得近乎固执：一旦某个报文丢失，后面即使收到完好的报文也一律不认，发送方要从丢失点起把之后发出去的全部重发一遍（go-back-N，回退 N 帧）。丢一个包，赔上整个在途窗口，这样的代价结构决定了 RDMA 无法容忍丢包；而在 UEC 到来之前，唯一的办法就是让有损网络模拟出无损行为，PFC 反压止损、ECN 发信号、DCQCN 调速率，三件套让 RoCEv2 在成本敏感的场景里跑了起来，这是完全理性的工程决策。

但代价同样有目共睹，序言里称之为"调参地狱"。PFC 是 IB credit 背压的粗粒度版本，暂停的不是一条虚通道而是整个优先级，会形成暂停风暴甚至死锁。这粗颗粒度不是设计失误，是有损基因模拟无损必须支付的代价：PFC 是事后刹车，暂停帧发出后在途数据仍会继续抵达，接收方要为每个无损优先级预留 headroom 缓冲，尺寸随带宽和链路距离线性增长，而片上缓冲恰恰是最昂贵的资源，无损等级经济上通常只开得起一两个；credit-based 是事前授权，字节离开源头前接收方缓冲已被记账，不存在"在途意外"，逐 VL 细颗粒度是天然产物。一个把缓冲状态做成协议原语，一个把刹车追加在自由发送的链路上，基因上天壤之别。更微妙的是：**IB 的无损是原生的，一致性由架构保证；RoCE 的无损是拼装的，一致性靠运维保证。** 用打补丁的方式追赶原生设计，追到最后往往比原生更复杂。

UEC 会让这套补丁体系失去存在的前提。UET（Ultra Ethernet Transport）的选择性重传（Selective Retransmission）把丢包的恢复代价从"整个窗口回退"压缩到"单个报文"，packet trimming 让交换机在拥塞时只留报头继续转发，而非原地静默丢弃。[12] 当端点能以线速消化丢包，网络就不再需要承诺不丢包。但要分清楚，**UEC 消灭的是"网络强制无损"，不是拥塞控制本身**，UET 自带全新拥塞控制，只是从"交换机反压链条"换成了"端点间协商"。[12] UEC 在链路层保留的可选项 Credit-Based Flow Control，是对 FC 和 IB 这套机制的一次坦率致敬：逐虚通道的信用授权，拥塞被扼杀在源头而非事后反压，落地难度比今天三件套的耦合调参简单许多。[13]

一个值得玩味的历史细节：PFC 本就不是为 RDMA 而生，它是 DCB 体系为 FCoE 打造的零件，是上一次让以太网模拟无损、承载光纤通道语义的尝试。FCoE 没有走远，这个零件却被 RoCE 捡了起来，在 AI 时代续写了同一个命题。这就是它的定位：不体面，但必要，撑起一个过渡时代，然后被为这个时代量身设计的东西替代。对它们的正确态度是尊敬，而不是留恋。

---

## 五、以太网的基因没有变，靠上层能力在补短板

最后说一说"以太网的选路能力"。

传统以太网的二层是"傻"的：广播、泛洪、生成树，选路能力约等于零。这个傻是设计出来的，是简单性的代价，也是简单性的来源。真正的选路智能安排在三层，子网内部则听天由命，这个分工在南北向流量主导的时代运转良好。

AI fabric 打破了这个平衡。集合通信的流量矩阵稀疏而巨大，少数几条大象流就能塞满链路；ECMP 基于流的哈希在这种场景下频繁极化，多条大流被哈希到同一条路径，其余路径空转。本书第十四章讲过 IB 对此的回答：自适应路由，交换机持续监测链路实时负载和队列深度，动态避开拥塞。这是一套闭环机制。传统以太网体系里，子网内部不存在对应的机制。

Packet spraying 是以太网世界的回答，但要说清楚它解决到了哪一步。UET 让发送端逐包变换 entropy 值（即 ECMP 哈希的输入字段，实践中通常是逐包改写 UDP 源端口号），把报文均匀打散到所有等价路径上，接收端负责重组。[14] 这确实比传统 ECMP 前进了一步：不再是一条大流固定占死一条路径，静态热点消失了。

但这依然不是 IB 的 adaptive routing，差别不是程度，是种类：adaptive routing 是闭环的，交换机拿实时负载做决策，哪条路径堵了就避开哪条；packet spraying 是开环的，发送端只管打匀，对任何一条路径此刻是否拥塞完全不感知。均匀分布不等于负载感知分布：它补的是 ECMP 静态哈希造成的热点，补不了实时拥塞下的动态避让。

更值得留意的是补偿发生的位置：这个能力实现在发送端网卡上，而不是二层交换机本身。以太网的二层交换设备没有变聪明，还是那套广播、泛洪、转发的基因；真正做事的是端点在报文头部动了一个字段。单靠传统的三层选路，已经无法满足 AI 流量对 fabric 内部路径利用率的要求，但以太网面对这个欠缺，并没有直接对二层那套简单笨拙的基因动手术，而是在它两端加一层东西兜底。补上这个缺口的是端点侧的工程手段，不是以太网二层本身获得了 IB 那种原生的闭环选路智能。

---

## 结语：螺旋

写到这里，可以把全书的线索收拢成一个图景了。

网络技术史上存在一个反复摆动的钟摆：一端是以 FC、IB 为代表的集中式确定性保证，网络对一切负责；另一端是以太网 + TCP 所代表的分布式尽力而为加端点智能，网络只管转发。每一轮摆动的方向，由当时的硬件成本曲线决定：当端点算力昂贵，把复杂性做进网络是理性的；当端点算力便宜，把复杂性收回端侧是理性的。过去四十年，摩尔定律持续把成本曲线推向端点一侧，所以钟摆的每一次回摆，最终都落在了"有损 fabric 加端侧可靠"这边。

但这不是简单的循环，而是螺旋。以太网 + TCP 拿下通用连接的那一轮，端侧可靠靠的是 CPU 上的软件协议栈；这一轮 UEC 对 IB 与 RoCEv2 的挑战，端侧可靠靠的是网卡 ASIC 里的硬件传输引擎。同一个端到端原则，在高出一个数量级的性能水平上被重新验证。每一次"回归"，都在更高的层次上重新出发。

无损 fabric 没有消失过。只要世界上还存在愿意为确定性支付溢价的场景，比如 HPC 的紧耦合仿真、金融的微秒竞争、frontier lab 对每一分 MFU（Model FLOPs Utilization，模型算力利用率）的偏执、企业关键业务存储对零丢包零抖动近乎偏执的稳定性追求（FC 至今仍是银行核心交易系统、大型数据库存储这类场景的事实标准），它就有自己的生存空间。这是过去四十年反复验证过的逻辑。

同样从历史逻辑出发，我的判断是：从工程复杂度、成本与规模这三个变量看，有损 fabric 加端侧可靠，是多数场景下更理性的选择。那么，为什么还要花一整本书去讲 InfiniBand？

因为 IB 把"网络承担确定性"这条技术路线，做到了教科书级的完备：从 credit 流控到集中式 SM，从 VL 仲裁到在网计算，它是这个方向上最纯粹、最自洽的样本。以我们中华文化的根基来举例：如果只读儒家一个流派的经典，那么读到的大概和圣经、佛经这些教人向善的东西没什么本质区别。你只有同时看懂了法家、道家、墨家这些儒家的对立面和侧面，才知道儒家在应对什么、拒绝什么、悄悄借用了什么。以太网这条线也是同样的道理：不先看懂 IB 把"网络承担确定性"这条路走到极致是什么样子，你就读不出 RoCEv2 在躲避什么、UEC 在偷师什么、SRD 又在反对什么。他山之石，可以攻玉，而 IB 正是这块最完整的他山之石。具体的技术会不断过时，但过去四十年这架钟摆摆动的结构，是可以从历史里读出来的规律。

这就是这本书真正想交付的东西。

谢谢你读到这里。

---

## 参考资料

以下为后记中引用的数据与结论的公开来源，按正文中出现的先后顺序排列，供延伸阅读与查证。所有链接访问于 2026 年 7 月。

1. **NVIDIA 网络业务规模**：季度网络收入近百亿美元、同比增长逾两倍；Spectrum-X 年化收入破百亿；IDC 数据显示 2026 年第一季度 NVIDIA 成为最大数据中心以太网交换机厂商
   AInvest, _The $6.9 Billion Acquisition That Turned Nvidia Into a Networking Giant_ — <https://www.ainvest.com/news/6-9-billion-acquisition-turned-nvidia-networking-giant-2606/>
   SDxCentral, _Nvidia: Networking is booming but your networks cost nothing_ — <https://www.sdxcentral.com/news/nvidia-networking-is-booming-but-your-networks-cost-nothing/>
   Windows News, _NVIDIA Becomes Largest Data Center Ethernet Switch Vendor in Q1 2026, IDC Says_ — <https://windowsnews.ai/article/nvidia-becomes-largest-data-center-ethernet-switch-vendor-in-q1-2026-idc-says.431907>

2. **微软为 OpenAI 打造的 IB 训练超算**：Eagle（14400 颗 H100，NDR InfiniBand，2023 年 Top500 第三）；全球首个大规模 GB300 NVL72 集群（Quantum-X800 InfiniBand + SHARP v4）
   Microsoft Azure Blog, _NVIDIA GB300 NVL72: Next-generation AI infrastructure at scale_ — <https://azure.microsoft.com/en-us/blog/microsoft-azure-delivers-the-first-large-scale-cluster-with-nvidia-gb300-nvl72-for-openai-workloads/>
   Microsoft Tech Community, _Annual Roundup on AI Infrastructure Breakthroughs for 2023_ — <https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/annual-roundup-of-ai-infrastructure-breakthroughs-for-2023/4097737>

3. **xAI Colossus**：十万颗（后扩至二十万颗）Hopper GPU，Spectrum-X 以太网 + BlueField-3 SuperNIC；"2024 年初约九成 AI 集群用 IB"的行业口径
   NVIDIA Newsroom, _NVIDIA Ethernet Networking Accelerates World's Largest AI Supercomputer, Built by xAI_ — <https://nvidianews.nvidia.com/news/spectrum-x-ethernet-networking-xai-colossus>
   The Register, _xAI's 100,000 H100 Colossus is glued together using Ethernet_ — <https://www.theregister.com/2024/10/29/xai_colossus_networking/>

4. **Meta 双 24576 卡对照集群**：一个 RoCE（Arista 7800），一个 Quantum-2 InfiniBand；两者调优后性能相当，最大 Llama 3 模型在 RoCE 集群训练
   Meta Engineering, _Building Meta's GenAI Infrastructure_ — <https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/>
   Meta Engineering, _How Meta trains large language models at scale_ — <https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/>

5. **OpenAI MRC**：基于 RoCE、吸收 UEC 技术并叠加 SRv6 源路由的训练网络方案，已部署于 OCI Abilene 与微软 Fairwater 的 GB200 训练超算
   OpenAI, _Supercomputer networking to accelerate large scale AI training_ — <https://openai.com/index/mrc-supercomputer-networking/>

6. **推理集群的 RoCE 经济优势**：IB 每端口成本高出 1.5–2.5 倍、推理负载原子化特征、RoCEv2 三层可路由性对边缘部署的适配
   NADDOD, _Training vs Inference: Why Your AI Network Architecture Needs to Be Different_ — <https://naddod.medium.com/training-vs-inference-why-your-ai-network-architecture-needs-to-be-different-97a5718f14aa>

7. **通信计算比决策阈值**：高于 25% 评估 IB、低于 10% 留在以太网；IB 与 RoCEv2 的 fabric 成本摊销对比
   Spheron, _GPU Networking for AI Clusters: InfiniBand vs RoCE vs Spectrum-X Decision Guide (2026)_ — <https://www.spheron.network/blog/gpu-networking-infiniband-roce-spectrum-x-guide/>

8. **Project Rainier**：近五十万颗（后超百万颗）Trainium2 芯片，第三代 EFA petabit 级互联，NeuronLink（域内）+ EFA（域间/跨数据中心）两级架构
   Data Center Dynamics, _AWS activates Project Rainier cluster of nearly 500,000 Trainium2 chips_ — <https://www.datacenterdynamics.com/en/news/aws-activates-project-rainier-cluster-of-nearly-500000-trainium2-chips/>
   Amazon, _AWS's Project Rainier: the world's most powerful computer for training AI_ — <https://www.aboutamazon.com/news/aws/aws-project-rainier-ai-trainium-chips-compute-cluster>

9. **Anthropic 硬件战略**：超百万颗 Trainium2 芯片训练与运行 Claude；同时使用 Trainium / TPU / NVIDIA GPU 的多元硬件战略
   Anthropic, _Anthropic and Amazon expand collaboration for up to 5 gigawatts of new compute_ — <https://www.anthropic.com/news/anthropic-amazon-compute>
   Anthropic, _Anthropic expands partnership with Google and Broadcom for multiple gigawatts of next-generation compute_ — <https://www.anthropic.com/news/google-broadcom-partnership-compute>

10. **EFA / SRD**：AWS 上唯一的 RDMA fabric（不提供 IB 与 RoCE）；OS-bypass 架构、网卡侧可靠传输、Trn2 实例 3.2 Tbps EFAv3 规格
    NVIDIA Dynamo Documentation, _EFA (RDMA over AWS Fabric) on EKS_ — <https://docs.nvidia.com/dynamo/kubernetes-deployment/cloud-provider-guides/aws/efa-rdma-over-aws-fabric>
    AWS, _Elastic Fabric Adapter (EFA)_ 官方文档 — <https://aws.amazon.com/hpc/efa/>

11. **SemiAnalysis 独立评估**：EFA 性能仍落后于 InfiniBand、Spectrum-X 及主流 RoCEv2 方案；Trainium2 的内存带宽/TCO 优势分析
    SemiAnalysis, _Amazon's AI Resurgence: AWS & Anthropic's Multi-Gigawatt Trainium Expansion_ — <https://newsletter.semianalysis.com/p/amazons-ai-resurgence-aws-anthropics-multi-gigawatt-trainium-expansion>

12. **UET 机制**：选择性重传、packet trimming、发送端窗口式与接收端 credit 式双拥塞控制机制
    Ultra Ethernet Consortium, _Ultra Ethernet Specification v1.0_ — <https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UE-Specification-6.11.25.pdf>
    Ultra Ethernet Consortium, _Ultra Ethernet Specification Update_ — <https://ultraethernet.org/ultra-ethernet-specification-update/>

13. **UE 链路层可选扩展**：Credit-Based Flow Control、Link Layer Retry、Packet Trimming 的机制细节
    Tom's Hardware, _Ultra Ethernet: The data-center interconnection of tomorrow detailed_ — <https://www.tomshardware.com/networking/ultra-ethernet-the-data-center-interconnection-of-tomorrow-detailed>
    Arista, _Demystifying Ultra Ethernet_ — <https://blogs.arista.com/blog/demystifying-ultra-ethernet>

14. **UE 架构设计原则**：逐包多路径（packet spraying）、有损运行以规避队头阻塞、端点侧乱序重组
    Hoefler et al., _Ultra Ethernet's Design Principles and Architectural Innovations_（arXiv 预印本）— <https://arxiv.org/pdf/2508.08906>
    HPCwire, _Ultra Ethernet Has Arrived: One Network to Rule Them All?_ — <https://www.hpcwire.com/2025/09/09/ultra-ethernet-has-arrived-one-network-to-rule-them-all/>

15. **端到端原则**
    J. H. Saltzer, D. P. Reed, D. D. Clark, _End-to-End Arguments in System Design_, ACM Transactions on Computer Systems, Vol. 2, No. 4, 1984.

> 注：第三方分析与媒体报道中的具体数字（收入、成本倍数等）多为各机构基于公开信息的统计或估算，不同口径间可能存在差异，正文已尽量采用区间或量级表述。厂商财务数据以官方财报（SEC 备案文件）为准。
