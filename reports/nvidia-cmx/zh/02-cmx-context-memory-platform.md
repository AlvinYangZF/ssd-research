# NVIDIA CMX（上下文内存平台）：深度技术架构分析

## 1. 引言 --- KV Cache 问题：为什么 AI 推理需要全新的内存层级

从单轮对话交互向长上下文推理和多步骤智能体工作流的转变，暴露了 AI 推理中一个根本性的基础设施瓶颈：KV Cache（键值缓存）。大语言模型（LLM）处理的每个 Token 都会生成键（Key）和值（Value）张量，这些张量必须保留以供后续注意力计算使用。随着上下文窗口从 8K 扩展到 128K，再到如今的 100 万+ Token，加之智能体 AI 系统在单次会话中串联数十个推理步骤，KV Cache 已从微不足道的簿记开销演变为 GPU 显存的头号消耗者 --- 其内存占用往往超过模型权重本身。

这不是仅靠更优算法就能解决的软件优化问题，而是一个存储架构问题。GPU HBM（高带宽内存）价格昂贵、功耗高、容量受限。现有的卸载策略 --- 溢出到主机 DRAM 或转存到通用共享存储 --- 会引入延迟断崖，导致 GPU 解码流水线停顿，严重损害 Token 吞吐量。业界需要一个位于 GPU HBM 和企业存储之间的专用内存层级：它必须理解 KV Cache 数据的临时性、块结构化特征，并能以解码流水线所要求的带宽和延迟提供服务。

NVIDIA 的解决方案是 **CMX（上下文内存存储平台）**，于 GTC 2026 发布，是更广泛的 STX（Storage）架构的首个机柜级实现。CMX 引入了一个全新的"G3.5"层级 --- 由 BlueField-4 DPU 驱动的以太网直连闪存存储 --- 专为 Pod 级 KV Cache 的存储、共享和检索而设计。本报告从存储工程师视角，深入剖析 CMX 的架构、数据流、性能指标及竞争定位。

---

## 2. KV Cache 挑战

### 2.1 什么是 KV Cache？

在 Transformer 推理过程中，注意力机制需要访问所有已处理 Token 的键（K）和值（V）投影。推理引擎不会在每个生成步骤重新计算这些投影，而是将其缓存在 GPU 显存中。Transformer 的每一层都会为序列中的每个 Token 生成各自的 K 和 V 张量，这些张量在该序列活跃推理期间必须全部保留。KV Cache 的大小随上下文长度和模型深度（层数和注意力头数）线性增长，且必须在纳秒到微秒级延迟内保持可访问，否则解码阶段将发生停顿。

单个 Token 的 KV Cache 大小计算公式约为：

```
KV cache per token = 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
```

以 Llama 3.1-70B 为例，该模型拥有 80 层、每层 8 个 KV 头、128 维头维度，采用 FP16（2 字节），每个 Token 消耗约 2 x 80 x 8 x 128 x 2 = 327,680 字节（约 320 KB）的 KV Cache。在 128K Token 上下文长度下，单个请求约需 40 GB --- 仅服务一个用户。

内存开销因模型而异，十分可观：

| 模型 | 上下文长度 | 单请求 KV Cache | 备注 |
|---|---|---|---|
| Llama 3.1-70B | 128K tokens | ~40 GB HBM | 单用户，FP16 |
| 70B 级模型 | 8K tokens | 每请求 ~20 GB | 32 并发约 ~640 GB |
| 70B 级模型 | 1M tokens | 每用户 ~15 GB（量化后） | 采用激进 KV 量化 |

Sources: [Pure Storage](https://blog.purestorage.com/perspectives/cut-llm-inference-costs-with-kv-caching/), [Introl](https://introl.com/blog/long-context-llm-infrastructure-million-token-windows-guide), [TensorWave](https://tensorwave.com/blog/estimating-llm-inference-memory-requirements)

在生产环境的并发水平下（数百到数千个同时会话），KV Cache 需求可在整个 GPU Pod 内达到 **数十 TB**。单块 NVIDIA Blackwell B200 GPU 拥有 192 GB HBM3e，即使一个包含 72 块 GPU 的机柜（NVL72 配置），总计也仅提供约 14 TB 的聚合 HBM。当多轮智能体在众多推理步骤间维持持久上下文时，这些容量会被迅速耗尽。

### 2.2 当前方案及其局限性

**仅使用 GPU HBM：** 性能最高的选项，但容量严格受限。运营商必须在支持更少的并发用户或截断上下文窗口之间做出取舍 --- 两者都会降低推理服务的质量和经济性。

**CPU DRAM 卸载：** 主机系统内存（DDR5/LPDDR5）每节点提供 1--2 TB 容量，单位容量成本低于 HBM。然而，GPU 与主机内存之间的 PCIe 总线引入了微秒级延迟，且带宽远低于 HBM。NVIDIA 的基准测试表明，通过 PCIe 进行 CPU-GPU 内存共享可以加速 KV Cache 卸载，但这仍然是节点本地方案，无法跨 Pod 共享。[NVIDIA Technical Blog](https://developer.nvidia.com/blog/accelerate-large-scale-llm-inference-and-kv-cache-offload-with-cpu-gpu-memory-sharing/)

**本地 NVMe SSD（G3 层级）：** 本地 SSD 以低成本提供 TB 级容量，但仅限于节点本地访问。存储在某一节点 SSD 上的 KV Cache 无法被其他节点的 GPU 高效访问。这导致"搁浅容量"问题 --- 一个节点的 SSD 上可能有大量温热 KV Cache，而处理相关请求的另一个节点却不得不从头重新计算相同上下文。Vrije Universiteit Amsterdam 的研究对 KV Cache 卸载到 NVMe SSD 的 I/O 模式进行了分析，发现虽然顺序写入吞吐量尚可，但解码阶段的读取模式在 SSD 控制器同时处理其他会话写入时，可能出现较高的尾延迟。[CHEOPS 2025 Workshop](https://atlarge-research.com/pdfs/2025-cheops-llm.pdf)

**共享企业存储（G4 层级）：** 传统 NAS/对象存储（NetApp、VAST、WEKA 等）提供 PB 级共享容量并具备数据保护能力，但毫秒级访问延迟使其不适用于活跃推理上下文。它为持久化数据而设计 --- 检查点、数据集、日志 --- 而非推理所需的临时性、高周转 KV 块。

**CXL 直连内存：** CXL（Compute Express Link）内存池化是一种新兴替代方案，可提供对共享 DRAM 池的缓存一致性访问。XConn Technologies 和 MemVerge 的研究表明，在卸载场景中相比基于 SSD 的 KV Cache 方案可实现超过 5 倍的性能提升。CXL 4.0 的目标是跨池化内存实现 1.5 TB/s 聚合带宽。然而，CXL 基础设施对大多数部署场景仍处于预生产阶段，需要专用交换机，且尚未在以太网上实现 Pod 级规模运行。[XConn/MemVerge](https://www.prweb.com/releases/xconn-technologies-and-memverge-to-deliver-breakthrough-scalable-cxl-memory-solution-to-offload-kv-cache-and-prefilldecode-disaggregation-in-ai-inference-workloads-302616566.html), [Introl CXL Guide](https://introl.com/blog/cxl-4-0-infrastructure-planning-guide-memory-pooling-2025)

差距显而易见：目前不存在一个生产就绪、Pod 级规模、针对 KV Cache 特定访问模式优化的共享内存层级。这正是 CMX 旨在填补的空白。

---

## 3. CMX 架构

### 3.1 NVIDIA AI 工厂内存层级体系

NVIDIA 为 AI 推理基础设施制定了五级内存与存储层级体系。理解这些层级对于准确定位 CMX 至关重要：

| 层级 | 介质 | 作用范围 | 延迟级别 | 角色 |
|---|---|---|---|---|
| **G1** | GPU HBM | 单 GPU | 纳秒级 | Token 生成中的活跃 KV Cache |
| **G2** | 系统 DRAM | 单节点 | 微秒级 | HBM 溢出 KV 的暂存/缓冲 |
| **G3** | 本地 NVMe SSD | 单节点 | ~100 us | 温热 KV Cache，短时间复用 |
| **G3.5** | **CMX（以太网直连闪存）** | **Pod 级** | **~100--500 us** | **共享 KV Cache，跨节点上下文复用** |
| **G4** | 共享企业存储 | 集群/数据中心 | 毫秒级 | 冷数据、检查点、持久化数据 |

Source: [NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [Blocks and Files](https://www.blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/4090541)

CMX 占据 G3.5 层级：它提供 **Pod 级共享的以太网直连闪存容量**，专为 KV Cache 的临时性、块结构化访问模式而构建。与 G3（节点本地 SSD）不同，CMX 使 KV 块可被 Pod 内任意 GPU 访问。与 G4（共享存储）不同，CMX 针对亚毫秒级访问进行了优化，并采用硬件加速数据完整性校验，而非企业级数据保护语义。

### 3.2 CMX 节点组成

每个 CMX 节点围绕三大核心组件构建：

**BlueField-4 存储处理器。** BlueField-4 是 CMX 的计算核心，集成了：

- **64 核 NVIDIA Grace CPU**（Arm Neoverse V2），核心数为 BlueField-3 16 核 Arm Cortex-A78 的 4 倍，算力约为前代的 6 倍
- **ConnectX-9 SuperNIC**，提供 **800 Gbps** 以太网吞吐（相比 BlueField-3 的 400 Gbps 翻倍）
- **128 GB LPDDR5** 内存和 **114 MB 共享 L3 缓存**，用于元数据、索引和 DMA 暂存
- **512 GB 板载 SSD**，用于本地固件和管理
- 硬件加速引擎，在 **800 Gbps 线速** 下实现加密和 CRC 数据保护，确保数据完整性且无 CPU 开销

Source: [Tom's Hardware](https://www.tomshardware.com/tech-industry/nvidia-launches-bluefield-4-stx-storage-architecture-for-agentic-ai), [ServeTheHome](https://www.servethehome.com/nvidia-bluefield-4-with-64-arm-cores-and-800g-networking-announced-for-2026/)

**NVMe SSD。** BlueField-4 直接管理连接的 NVMe 驱动器，处理数据放置、磨损均衡协调和 I/O 调度。SSD 作为 KV Cache 块的持久化（但生命周期短暂）后端存储。NVIDIA 未指定每个 CMX 节点的固定驱动器数量，这取决于系统供应商的机箱设计。来自 Supermicro、QCT 和 AIC 的参考平台预计将支持高密度 NVMe 配置。有关 SSD 供应商就绪度和与 CMX 相关的驱动器级规格的深入分析，请参见 [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md)。

**Spectrum-X 以太网交换架构。** CMX 节点通过运行 RDMA over Converged Ethernet（RoCE，融合以太网远程直接内存访问）的 NVIDIA Spectrum-X 交换机连接到 GPU 计算 Pod。这为 KV Cache 传输提供了低延迟、高带宽的数据通路，且无需承担 InfiniBand 的成本和复杂性。Spectrum-X 的自适应路由和拥塞控制针对 KV Cache 读取特有的突发性小块 I/O 模式进行了调优。选择以太网而非 InfiniBand 具有重要战略意义：它允许 CMX 部署在已标准化以太网交换基础设施的数据中心中，而这涵盖了绝大多数云和企业环境。Spectrum-X 在标准以太网之上增加了 RDMA 能力和无损流控，在 Pod 内提供 InfiniBand 级性能，无需更换网络架构技术。

### 3.3 软件栈

CMX 软件栈由三个紧密衔接的层级组成：

**DOCA Memos。** 这是针对 BlueField-4 和 CMX 优化的 SDK，暴露 **简洁的键值 API** 用于 KV Cache 管理、共享和放置。DOCA Memos 将上下文缓存视为一等资源类型，充分利用 KV 块和推理模式的独特属性。它在规模化场景下处理 KV Cache 的路由和复用，使推理应用保持无状态，而由 CMX 管理底层存储。DOCA Memos 提供：

- KV 块的 put/get/delete 操作
- 基于哈希的块寻址，支持去重和查找
- 安全隔离访问，配合硬件加速的完整性校验和加密
- KV 块生命周期、所有权和共享策略的元数据管理

Source: [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-bluefield-4-powers-new-class-of-ai-native-storage-infrastructure-for-the-next-frontier-of-ai)

**NIXL（NVIDIA Inference Transfer Library，NVIDIA 推理传输库）。** 一个开源的高性能库，用于加速异构内存和存储层级之间的点对点数据移动。NIXL 提供传输层，在 GPU HBM（G1）、系统 DRAM（G2）、本地 SSD（G3）、CMX（G3.5）和共享存储（G4）之间搬运 KV 块。它将底层传输方式 --- 无论是 PCIe、NVLink 还是 RoCE --- 抽象为统一的传输 API。[NVIDIA NIXL](https://blockchain.news/news/nvidia-nixl-open-source-inference-transfer-library)

**NVIDIA Dynamo。** 分布式推理服务框架，编排整个流水线。Dynamo 与 CMX 相关的核心组件包括：

- **KV 感知路由器（KV-Aware Router）：** 评估传入请求与 Pod 内现有 KV Cache 状态的匹配程度。它根据 KV Cache 重叠率（缓存命中率）、预填充代价（需计算的新 Token 数）和解码代价（活跃块数）计算将请求路由到各 Worker 的开销，从而最小化集群内的冗余 KV 重算。[NVIDIA Dynamo Docs](https://docs.nvidia.com/dynamo/latest/user-guides/kv-cache-aware-routing)
- **KV 块管理器（KV Block Manager）：** 一个成本感知的缓存引擎，决定何时以及在何处跨内存层级传输 KV 块。它处理从 G1 到 G2/G3.5 的驱逐、从 G3.5 回升至 G1 的提升，以及过期上下文的垃圾回收。
- **SLO 规划器（SLO Planner）：** 监控容量利用率和预填充活动，动态调整 GPU 资源分配以满足延迟服务等级目标。
- **预填充/解码分离（Disaggregated Prefill/Decode）：** 将计算密集型的预填充阶段（处理输入提示）与内存密集型的解码阶段（生成输出 Token）分离，使两者分别在最优配置的硬件上调度。

Source: [NVIDIA Dynamo Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)

### 3.4 Pod 级上下文共享

CMX 相对于节点本地 SSD 存储（G3）的核心架构差异在于 **Pod 范围的共享访问**。Pod 内任何 GPU 节点均可通过 Spectrum-X RoCE 交换架构对任意 CMX 节点进行 KV 块的读写。这实现了几项关键能力：

- **跨节点 KV 复用：** 若用户 A 的会话为一份 500K Token 的文档生成了 KV Cache，当用户 B 对同一文档提交查询时，KV Cache 可直接从 CMX 提供给用户 B 的 GPU，无需重新计算。
- **智能体状态共享：** 在智能体 AI 工作负载中，多个专业化智能体可能需要访问相同的推理上下文。CMX 提供共享命名空间，智能体可直接读取彼此的 KV 状态，无需在 GPU 节点之间进行点对点传输。
- **负载均衡驱逐：** 当某个 GPU 节点的 HBM 面临压力时，其 KV 块可被驱逐到 CMX，随后由被分配接续该会话的另一节点检索。

CMX 为每个 GPU Pod 提供 **PB 级共享容量**，相比 HBM 的数十 TB 聚合容量实现了质的飞跃。[NVIDIA CMX Product Page](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/)

---

## 4. 数据流

### 4.1 请求到达与 KV Cache 查找

当推理请求到达 Dynamo 服务层时，执行以下流程：

1. **提示词哈希与 KV 索引查找。** Dynamo KV 感知路由器计算传入提示词 Token 序列（或其前缀片段）的哈希值，并在分布式 KV Cache 索引中进行查找。该索引追踪已有的 KV 块、其所在位置（G1、G2、G3、G3.5）及新鲜度。

2. **缓存命中路径（完全或部分命中）。** 若找到匹配的 KV 块：
   - 位于 G1（同一 GPU 的 HBM）的块：直接使用，零传输开销。
   - 位于 G2（同一节点的主机 DRAM）的块：通过 PCIe/NVLink 传输至 G1，微秒级延迟。
   - 位于 G3.5（CMX）的块：NIXL 通过 Spectrum-X 交换架构发起 RDMA 读操作，从 CMX 节点的 NVMe 支撑缓冲区读取数据至目标 GPU 的 HBM。CMX 节点上的 BlueField-4 负责 NVMe 读取、CRC 校验和 RDMA 传输。
   - 对于已缓存的前缀，预填充阶段被缩短甚至完全跳过。NVIDIA 报告称，从存储获取 KV Cache 相比重新计算，**TTFT（首 Token 延迟）提升可达 20 倍**。[VAST Data](https://www.vastdata.com/blog/how-nvidia-dynamo-vast-unlock-context-reuse-at-scale)

3. **缓存未命中路径。** 若未找到匹配的 KV 块：
   - 请求被路由至预填充 Worker，从输入 Token 计算 KV Cache。
   - 新计算的 KV 块写入 G1（活跃使用），并异步写入 G3.5（CMX）以供后续复用。

### 4.2 写入路径：从 GPU HBM 到闪存

KV Cache 写入 CMX 的流程如下：

1. **驱逐触发。** GPU 节点上的 KV 块管理器判定 HBM 压力需要驱逐冷 KV 块，或新生成的 KV 块应持久化到 CMX 以供共享。
2. **NIXL 传输。** NIXL 从 GPU HBM（或主机 DRAM 暂存缓冲区）向目标 CMX 节点发起 RDMA 写操作，写入定向至特定 BlueField-4 端点。
3. **BlueField-4 处理。** CMX 节点上的 BlueField-4 接收 RDMA 负载，以线速（800 Gbps）执行 CRC 完整性校验，可选地对数据加密，然后向连接的 SSD 下发 NVMe 写命令。
4. **索引更新。** KV 块的元数据（哈希、位置、时间戳、所有权）在分布式索引中更新，以便后续查找能定位该块。
5. **确认应答。** CMX 节点向 GPU 节点确认写入完成，GPU 节点即可释放相应 HBM 容量。

从存储视角来看，写入路径设计为相对于解码流水线**异步且非阻塞**的。GPU 不会等待 NVMe 写入完成后再继续 Token 生成。BlueField-4 的 LPDDR5 缓冲区吸收写入突发，NVMe 刷盘在后台完成。这一点至关重要，因为解码阶段对延迟极为敏感 --- 写入路径中的任何同步 I/O 都会直接降低 Token 生成吞吐量。

NVMe 写入模式以**大块顺序写入**为主（KV 块通常为数十 MB），与 SSD 控制器针对顺序工作负载的优化方向一致。但 KV 块的高周转率（一次写入、少量读取后，在数分钟到数小时内即被丢弃）会在闪存介质上造成较高的写放大。面向 CMX 工作负载的 SSD 厂商需要针对这种临时写入模式优化垃圾回收和预留空间策略。

### 4.3 预取与预测性暂存

CMX 架构中的一项关键优化是**预测性预取**。由于提示词和上下文在到达 LLM 之前会经过 Dynamo 聚合器，系统可以预判即将到来的解码步骤所需的 KV 块。聚合器向 CMX 存储引擎发送提示，将 KV 条目从 NVMe 预取到 BlueField-4 的 LPDDR5 暂存缓冲区，这样当 GPU 发起 RDMA 读取时，数据已在快速内存中就绪，无需等待 NVMe 读取。

该预取机制对维持解码吞吐量至关重要。解码阶段逐个生成 Token，对延迟极为敏感 --- 等待从 SSD 读取 KV 块的一次停顿即可传导为可感知的生成暂停。通过将 KV 块提前暂存到 BlueField-4 的 128 GB LPDDR5 缓冲区，CMX 将原本的同步 NVMe 读取（数十微秒）转化为快速内存读取（个位数微秒）。

### 4.4 多轮持久化

对于多轮对话和智能体工作流，CMX 提供**会话级 KV 持久化**：

- 每轮对话结束后，累积的 KV Cache 保留在 CMX 中而非被丢弃。
- 当用户（或智能体）发起下一轮对话时，Dynamo 路由器在 CMX 中定位该会话的 KV 块并预取至分配的 GPU。
- 这消除了每轮对话都需重新处理整个对话历史的需求 --- 该成本随对话长度线性增长，在长交互中可能主导预填充时间。

### 4.5 智能体上下文共享

在多智能体系统中，CMX 充当**共享上下文总线**：

- 智能体 A（例如研究型智能体）处理完文档语料后，将其 KV Cache 写入 CMX。
- 智能体 B（例如摘要型智能体）从 CMX 读取智能体 A 的 KV Cache，基于已有上下文继续工作，无需重新处理相同文档。
- 智能体 C（例如验证型智能体）读取智能体 A 和 B 的缓存，进行交叉验证。

这一模式消除了智能体间的冗余预填充计算，降低了每个智能体任务的总 GPU 时数。CMX 基于哈希的寻址方式支持细粒度共享 --- 智能体可共享单个 KV 块（对应特定文档片段），而非整个单体缓存。

### 4.6 生命周期管理与垃圾回收

CMX 中的 KV Cache 块本质上是临时的。与传统存储数据在显式删除前持久存在不同，KV 块具有天然的过期触发机制：

- **会话超时：** 当用户会话结束或空闲超过可配置阈值时，所有关联的 KV 块成为驱逐候选。
- **上下文窗口滑动：** 对于使用滑动窗口注意力的模型，对应于已超出注意力窗口的 Token 的 KV 块可被垃圾回收。
- **引用计数：** 当引用某个共享 KV 块的最后一个智能体或会话终止时，该块的引用计数降为零，成为可回收状态。
- **容量压力：** 当 CMX 闪存利用率超过高水位线时，系统根据综合策略驱逐块，该策略考虑最近访问时间、块大小和共享程度（被广泛共享的前缀块比单会话块保留更久）。

BlueField-4 的 Grace CPU 在本地处理这些生命周期管理，无需 GPU 计算节点参与。这将元数据簿记和垃圾回收从推理流水线中卸载出来，确保存储管理开销不会与 Token 生成竞争 GPU 或主机 CPU 周期。

从 SSD 磨损角度来看，高写入速率与短块生命周期的组合意味着 CMX 驱动器将承受远高于典型企业存储工作负载的每日全盘写入次数（DWPD）。NVIDIA 尚未公布 CMX 认证驱动器的目标 DWPD 规格，但鉴于其写入模式，额定 3+ DWPD（混合用途企业级 SSD 的典型值）是合理的起点。参与 STX 生态的 SSD 厂商 --- 包括 Solidigm、Samsung 和 Kioxia --- 预计将提供针对临时块访问模式调优垃圾回收的 CMX 优化固件配置。

---

## 5. 性能指标

### 5.1 核心数据

NVIDIA 声称 CMX/STX 相比传统基于 CPU 的存储架构具有以下优势：

| 指标 | CMX/STX 声称 | 对比基准 |
|---|---|---|
| Token 吞吐量 | **最高 5 倍 tokens/second** | 对比传统存储方案 |
| 能效比 | **最高 5 倍 tokens/watt** | 对比传统存储方案 |
| 能源效率 | **4 倍** | 对比基于 CPU 的存储（STX 整体） |
| 数据摄入 | **2 倍** | 对比基于 CPU 的存储（STX 整体） |
| 首 Token 延迟 | **最高 14x--20x 提升** | 对比从头重算 KV Cache |

Sources: [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption), [NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [VAST Data](https://www.vastdata.com/blog/how-nvidia-dynamo-vast-unlock-context-reuse-at-scale)

### 5.2 测试方法与注意事项

NVIDIA 发布的资料将 CMX 与"传统存储方案"和"基于 CPU 的存储架构"进行对比，但未明确指定具体基准配置（CPU 代数、SSD 型号、网络架构或软件栈）。这使得独立验证较为困难。然而，基于以下原因，这些声称在架构层面是合理的：

**Token 吞吐量（5 倍）。** 核心机制是**消除 GPU 解码停顿**。当 KV Cache 必须通过非优化路径从通用存储获取时，GPU 在传输期间处于空闲状态。CMX 的 RDMA 传输、BlueField-4 硬件卸载和预测性预取的组合，旨在确保 KV 块在解码步骤需要之前就已到达 G1。即使停顿频率的适度降低也会累积为巨大的吞吐量增益，因为解码阶段本质上是顺序执行的 --- 一次停顿会延迟所有后续 Token。

**能效比（5 倍）。** 这直接源于吞吐量提升。若 GPU 花在空闲等待 KV 数据和重算已有上下文上的时间减少，则相同数量的 Token 消耗更少的 GPU 时数。由于 GPU 在推理 Pod 中占据主要功耗，减少 GPU 空闲时间几乎线性地转化为更优的 tokens-per-watt。此外，BlueField-4 的硬件加速数据通路（加密、CRC、NVMe 管理）相比通用 x86 存储控制器上等效的 CPU 处理消耗的功率要低得多。

**TTFT 提升（14x--20x）。** 这是最直观的指标。从存储获取预计算的 KV Cache（即使是 NVMe 延迟）也远快于在 GPU 上重新执行预填充计算。对于 128K Token 的上下文，单 GPU 上的预填充可能需要数秒；从 CMX 获取对应的 KV Cache 仅需毫秒。

**这些声称未涉及的方面：** 5 倍数据很可能是在最大化 CMX 优势的条件下测得的 --- 长上下文、高 KV Cache 复用率，以及基准系统出现显著停顿的工作负载。对于上下文短、单轮交互且 KV Cache 完全能装入 HBM 的工作负载，CMX 不会带来吞吐量收益（但通过释放 HBM 用于更多并发会话，仍可能改善多用户扩展性）。

---

## 6. CMX 与替代方案对比

| 维度 | 仅 GPU HBM | CPU DRAM 卸载 | CXL 直连内存 | 通用共享存储（G4） | **NVIDIA CMX（G3.5）** |
|---|---|---|---|---|---|
| **作用范围** | 单 GPU | 单节点 | 单机柜（新兴） | 集群级 | **Pod 级** |
| **容量** | 192 GB/GPU | 1--2 TB/节点 | 数十 TB/机柜 | PB 级 | **PB 级/Pod** |
| **延迟** | ~ns | ~us (PCIe) | ~100--300 ns (CXL) | ~ms | **~100--500 us** |
| **带宽** | TB/s (HBM) | ~100 GB/s (PCIe) | ~1.5 TB/s (CXL 4.0) | ~数十 GB/s | **800 Gbps/节点 (100 GB/s)** |
| **共享性** | 无 | 无 | 有限（机柜） | 全集群 | **全 Pod** |
| **KV 感知** | 仅软件层 | 仅软件层 | 仅软件层 | 无 | **硬件 + 软件** |
| **数据保护** | 无 | 无 | 无 | 企业级 | **线速 CRC + 加密** |
| **生产就绪度** | 当前可用 | 当前可用 | 预生产阶段 | 当前可用 | **2026 下半年** |
| **单位 GB 成本** | $$$$$ | $$$ | $$$$ | $ | **$$** |

**CXL 值得特别关注**，它是 CMX 在 KV Cache 卸载领域最直接的架构竞争者。CXL 提供更低的延迟（亚微秒级）和缓存一致性访问，但需要 CXL 兼容的 CPU、GPU 和交换机，而这些尚未广泛部署。CXL 4.0 的内存池化能力在理论上极具吸引力，但其生态系统在生产就绪度上落后 CMX 1--2 年。CMX 选择标准以太网（通过 Spectrum-X 和 RoCE）是务实之举，充分利用了现有数据中心网络投资。

**通用存储厂商**（VAST Data、WEKA、Dell、NetApp）并未被 CMX 取代 --- 它们继续服务于 G4 层级。其中多家正在积极与 CMX 和 Dynamo 集成。VAST Data 的开源 VUA（VAST Undivided Attention）软件、WEKA 的 Augmented Memory Grid 以及 Dell 的 Project Lightning 均提供 KV Cache 卸载能力，可根据部署拓扑与 CMX 形成互补或竞争关系。有关存储厂商 KV Cache 策略和 SSD 就绪度的详细分析，请参见 [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md)。

---

## 7. 应用场景

### 7.1 长上下文推理

CMX 最直接的应用场景是服务具有 128K--1M+ Token 上下文窗口的模型。在这些上下文长度下，单个请求的 KV Cache 消耗 15--40+ GB HBM，严重限制并发能力。CMX 允许运营商将已完成预填充的 KV 块从 HBM 驱逐至 G3.5 层级，释放容量用于更多并发会话，同时保留为进行中解码快速检索上下文的能力。

**存储工程师视角：** I/O 模式的特征是大块顺序写入（KV 块从 HBM 驱逐，通常每块数十 MB），随后是大块顺序读取（为解码检索 KV 块）。写密集阶段发生在预填充期间及之后；读密集阶段发生在解码期间。每个会话的工作集大小与上下文长度成正比。规模化场景下 SSD 耐久度是关注点 --- KV 块是临时的（数分钟到数小时的生命周期），若 SSD 固件未针对此访问模式优化则会产生较高写放大。

### 7.2 多轮对话

生产级对话服务（客户支持、编程助手、研究工具）产生多轮会话，每轮基于累积的上下文构建。若无 CMX，此前各轮的 KV Cache 必须保留在 GPU HBM 中（规模化时成本高昂）或从对话文本重新计算（缓慢且浪费）。CMX 跨轮次保留会话的 KV Cache，实现长对话的亚秒级延续。

**存储工程师视角：** 该工作负载类似于具有分钟到小时级 TTL 的会话亲和性缓存。访问模式为会话内的一次写入多次读取，会话超时后整体驱逐。容量规划需考虑峰值并发会话数乘以平均会话 KV Cache 大小。

### 7.3 智能体 AI 工作流

智能体 AI 系统 --- 由编排器调度专业化子智能体在多步骤循环中进行推理、规划和执行 --- 代表了 CMX 最高价值的应用场景。每个智能体步骤生成的 KV Cache 可能为后续步骤所需。若无共享上下文存储，每个智能体必须在 HBM 中保留所有先前上下文（对长链路不切实际）或将先前输出作为文本重新摄入（丧失 KV 生成的计算投入）。

CMX 使智能体能够**将 KV 块写入共享命名空间**并**读取彼此缓存的上下文**，消除冗余计算。对于一个 10 步骤的智能体工作流，每步处理 100K Token 上下文，KV Cache 复用相比重算可实现 10 倍的 GPU 总算力缩减。

**存储工程师视角：** 智能体工作负载生成复杂的 KV 块依赖图，存在部分共享。存储系统必须支持细粒度的块级寻址（而非仅会话级）、并发读写，以及在智能体工作流完成或失败时对孤立块的垃圾回收。

### 7.4 多用户 KV 复用（前缀缓存）

许多推理场景涉及共享前缀 --- 系统提示词、RAG（检索增强生成）文档上下文或跨用户相同的通用指令集。CMX 实现了 **Pod 级前缀缓存**：共享前缀的 KV Cache 仅计算一次并存储在 CMX 中，随后为所有共享该前缀的后续请求提供服务。

对于一个面向同一文档语料服务 1,000 个并发用户的 RAG 密集部署，前缀缓存可消除超过 90% 的预填充计算，直接转化为 GPU 时数、功耗和首 Token 延迟的等比例降低。

**存储工程师视角：** 前缀缓存块具有高价值、长生命周期和读密集特征。它们应被固定在 BlueField-4 的 LPDDR5 暂存缓冲区或快速 SSD 层级以最小化访问延迟。去重是哈希寻址方案的内在特性 --- 相同的前缀映射到相同的 KV 块地址。

---

## 8. 关键要点

1. **CMX 是存储架构，而非单纯的软件功能。** 它在 AI 推理内存层级体系中引入了一个物理上独立的层级（G3.5），配备专用硬件（BlueField-4）、专用网络（Spectrum-X RoCE）和专用软件（DOCA Memos、NIXL）。存储工程师应将其视为与传统 SAN/NAS 并列的全新基础设施类别。

2. **BlueField-4 DPU 是核心组件。** 其 64 核 Grace CPU、800 Gbps 网络、128 GB LPDDR5 暂存缓冲区以及硬件加速 CRC/加密引擎，将原本的商用 NVMe 存储转变为具备 KV Cache 感知、推理优化的存储节点。DPU 在无需主机 CPU 参与的情况下同时处理控制面（块查找、放置、生命周期管理）和数据面（NVMe I/O、RDMA 传输、完整性校验）。

3. **SSD 选型至关重要。** CMX 的性能取决于 BlueField-4 后端的 NVMe 驱动器。KV Cache 工作负载的特征是高写入速率的临时数据（短块生命周期）、大块顺序 I/O，以及解码阶段的高读取吞吐。针对这些模式优化的 SSD --- 高耐久度、低尾延迟、高顺序吞吐 --- 将优于通用驱动器。包括 Solidigm、Samsung、Micron、Kioxia 和 SK hynix 在内的 SSD 厂商正在积极为此工作负载联合设计驱动器。详细厂商分析请参见 [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md)。

4. **Pod 级共享是相对节点本地 SSD 缓存的核心差异化优势。** 任何系统都能将 KV Cache 卸载到本地 NVMe。CMX 的价值主张在于通过高带宽以太网架构使该缓存可被 Pod 内每个 GPU 访问。这使前缀缓存、多轮持久化和智能体上下文共享成为可能 --- 这些功能均无法仅靠节点本地存储实现。

5. **5 倍性能提升是有条件的。** 该数据适用于长上下文、高 KV 复用率，以及基线系统采用通用存储的配置。上下文短、单轮交互且完全装入 HBM 的工作负载将获得极少收益。运营商在决定部署 CMX 前，应针对其特定工作负载组合对比现有基础设施进行基准测试。

6. **CXL 是长期架构竞争者。** CXL 直连内存池提供 CMX 无法匹配的更低延迟和缓存一致性。然而，CXL 基础设施在生产就绪度上落后 CMX 1--2 年。NVIDIA 的判断是以太网直连闪存（CMX）可在 2026 下半年规模化部署，而 Pod 级 CXL 内存池化仍是 2027--2028 年的故事。

7. **生态系统支持广泛。** 存储厂商（VAST、WEKA、Dell、NetApp、DDN、Hitachi、HPE、IBM）、系统 OEM（Supermicro、QCT、AIC）以及云服务商（CoreWeave、Crusoe、Lambda、OCI、Vultr、Nebius）均已公布 STX/CMX 采用计划。如此广泛的支持表明，至少在 CXL 替代方案成熟之前，CMX 将成为 NVIDIA 平台上推理上下文存储的事实标准。[NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption)

8. **软件栈与硬件同等重要。** Dynamo 的 KV 感知路由、NIXL 的异构传输抽象以及 DOCA Memos 的键值 API 共同决定了 CMX 硬件的实际利用效率。部署 CMX 的运营商应将 Dynamo 集成作为前提条件而非事后补充来规划。

9. **容量规划需要新模型。** 基于数据集大小和 IOPS 需求的传统存储容量规划不适用于 CMX。运营商需要建模：峰值并发会话数 x 每会话平均 KV Cache 大小 x 复用系数 x 保留时间窗口。例如，一个为 1,000 个并发用户服务 128K Token 上下文、使用 70B 模型的部署，产生约 40 TB 的活跃 KV Cache（1,000 x 40 GB）。在前缀缓存实现 50% 复用率、保留窗口为 30 分钟的情况下，CMX 层级大约需要 20 TB 可用闪存容量，另加写放大和垃圾回收开销的余量。

10. **命名演变彰显 NVIDIA 的战略意图。** CMX 最初在 CES 2026 以 ICMS（Inference Context Memory Storage，推理上下文内存存储）之名发布，后在 GTC 2026 更名为 CMX。"上下文内存"的定位刻意将其塑造为全新的基础设施品类 --- 既非传统存储也非传统内存 --- 强化了 NVIDIA 构建 AI 原生数据中心架构的愿景，其中每个层级都为推理工作负载量身定制。

---

*本报告撰写于 2026 年 3 月。CMX 平台预计将于 2026 下半年由 NVIDIA 合作伙伴推出，作为 Vera Rubin 基础设施部署的一部分。所有性能声称均基于 NVIDIA 公开发布的资料，截至撰写时尚未经独立验证。*
