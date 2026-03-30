# BlueField-4 与 DOCA Memos：NVIDIA CMX 的存储处理器核心

## 1. 引言

NVIDIA BlueField-4 代表了面向 AI（人工智能）工作负载的存储基础设施架构的根本性变革。前几代 DPU（数据处理器）侧重于通用基础设施卸载——网络、安全、虚拟化——而 BlueField-4 则专为推理阶段的 KV cache（键值缓存）管理而打造，定位为一款**存储处理器**。它是 NVIDIA CMX（Context Memory Extensions，上下文内存扩展）平台的硅芯片核心，能够将一个机架的 NVMe（非易失性内存主机控制器接口规范）SSD（固态硬盘）转变为共享的、Pod 级别的上下文内存层。

这一演进意义重大。BlueField-2 引入了可编程 DPU 的概念，能够从主机 CPU 卸载 SmartNIC（智能网卡）、存储和安全功能。BlueField-3 将这一愿景扩展到 400 Gb/s，配备更多 Arm 核心和硬件加速器，在云原生基础设施中站稳脚跟。BlueField-4 则走上了一条截然不同的路线：它集成了一颗完整的 NVIDIA Grace CPU（64 个 Arm Neoverse V2 核心），搭配 ConnectX-9 800 Gb/s 网络引擎，并增加了专门针对 KV cache 完整性、加密和 NVMe 管理优化的硬件加速引擎。其成果并非仅仅是一款更快的 DPU——而是一个全新品类的存储处理器，设计用于 GPU 内存与持久化闪存之间，管理 agentic AI（智能体式 AI）工作负载在规模化运行时所产生的临时上下文数据。

NVIDIA 于 2026 年 1 月在 CES 2026 上发布了 BlueField-4，随后于 2026 年 3 月在 GTC 2026 上推出了 BlueField-4 STX 参考架构。该芯片预计将于 2026 年下半年作为 Vera Rubin 平台家族的一部分进入早期可用阶段。对于存储工程师而言，BlueField-4 及其配套的 DOCA Memos SDK（软件开发套件）定义了一种全新的运营模型：专为 KV cache 构建的存储，通过键值 API 而非传统的块或文件抽象进行管理，由 NVIDIA Dynamo 等推理框架而非传统存储控制器进行编排。

本报告深入分析 BlueField-4 的硬件架构、存储处理器角色、DOCA Memos SDK 以及共同构成 CMX 存储平面的完整软件栈。

Sources: [NVIDIA Newsroom: BlueField-4 Powers New Class of AI-Native Storage](https://nvidianews.nvidia.com/news/nvidia-bluefield-4-powers-new-class-of-ai-native-storage-infrastructure-for-the-next-frontier-of-ai), [NVIDIA Blog: BlueField-4 AI Factory](https://blogs.nvidia.com/blog/bluefield-4-ai-factory/), [NVIDIA Newsroom: BlueField-4 STX Architecture](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption)

---

## 2. BlueField-4 硬件架构

### 2.1 处理器复合体：搭载 64 个 Arm Neoverse V2 核心的 Grace CPU

BlueField-4 的计算子系统基于 NVIDIA Grace CPU 构建，配备 64 个 Arm Neoverse V2 核心。相较于 BlueField-3 的 16 个 Arm Cortex-A78 核心，核心数量实现了 4 倍增长，同时还伴随着微架构的代际升级。Neoverse V2 核心支持 SVE2（可扩展向量扩展 2），提供更高的 IPC（每时钟周期指令数），并采用先进制程，使 BlueField-4 的总晶体管数达到 640 亿，约为 BlueField-3 220 亿晶体管的三倍。

NVIDIA 宣称其计算性能为 BlueField-3 的"6 倍"。对于存储工作负载而言，额外的计算余量至关重要：它能够支持更丰富的带内分析、实时遥测处理、更智能的机箱管理、预测性维护以及动态工作负载调度——所有这些都在 DPU 上运行，无需消耗主机 CPU 或 GPU 算力。

| 规格 | BlueField-3 | BlueField-4 |
|---|---|---|
| CPU 核心 | 16x Arm Cortex-A78 | 64x Arm Neoverse V2 |
| 晶体管数量 | 22 billion | 64 billion |
| 内存 | 32 GB DDR5 | 128 GB LPDDR5 |
| 缓存 | 8 MB L2 | 114 MB 共享 L3 |
| 板载 SSD | 128 GB | 512 GB |
| 网络引擎 | ConnectX-7 | ConnectX-9 |
| 网络带宽 | 400 Gb/s | 800 Gb/s |
| 主机接口 | PCIe Gen 5 x16 | PCIe Gen 6 x16 |
| 计算倍数 | 1x（基准） | 6x |

Sources: [HPCwire: NVIDIA Cranks BlueField-4 to 800 Gbps](https://www.hpcwire.com/2025/10/28/nvidia-cranks-its-bluefield-4-dpu-to-800-gbps/), [ServeTheHome: BlueField-4 64 Arm Cores](https://www.servethehome.com/nvidia-bluefield-4-with-64-arm-cores-and-800g-networking-announced-for-2026/), [Tom's Hardware: BlueField-3 and BlueField-4 DPUs](https://www.tomshardware.com/news/nvidia-unveils-bluefield-3-and-bluefield-4-dpus)

### 2.2 内存子系统

BlueField-4 集成了 128 GB LPDDR5 内存，相较于 BlueField-3 的 32 GB DDR5 实现了 4 倍增长。从 DDR5 到 LPDDR5 的切换反映了功耗效率目标以及在 800 Gb/s 线速下管理 KV cache 元数据的带宽需求。114 MB 共享 L3 缓存（相比 BlueField-3 的 8 MB L2）为 NVMe 控制器固件、DOCA 运行时服务以及加密/完整性状态机提供了充裕的工作集空间。

板载 512 GB SSD（从 128 GB 升级而来）作为启动设备和 DOCA 运行环境、容器镜像及遥测日志的本地暂存空间。这与 BlueField-4 在 CMX 机箱中管理的 NVMe SSD 不同——后者是通过存储背板连接的外部驱动器。

### 2.3 ConnectX-9 网络引擎：800 Gb/s 以太网与 RDMA

集成的 ConnectX-9 SuperNIC 引擎提供 800 Gb/s 的聚合网络带宽，是 BlueField-3 400 Gb/s 的两倍。它同时支持 InfiniBand 和以太网的 800 Gb/s 传输，其中 NVIDIA 的 Spectrum-X 以太网交换架构是 CMX 的主要部署目标。

对于存储工程师而言，关键能力在于 **RoCE（RDMA over Converged Ethernet，融合以太网 RDMA）**。CMX 数据节点通过 RDMA 与 GPU 计算节点通信，完全绕过主机 CPU。ConnectX-9 引擎在硬件中处理 RDMA 协议终结，包括 NVMe-oF（NVMe over Fabrics，NVMe 光纤通道）目标终结。这意味着 BlueField-4 可以通过网络交换架构呈现 NVMe 命名空间，无需在关键路径上进行基于软件的协议转换。

ConnectX-9 还支持从 Spectrum-X 继承的高级拥塞控制和自适应路由功能，这对于在多租户 Pod 环境中保持稳定、低抖动的 RDMA 性能至关重要——在这种环境中，数百个 GPU 节点可能同时竞争 KV cache 访问。

Sources: [NVIDIA ConnectX-9 SuperNIC Analysis](https://www.naddod.com/ai-insights/nvidia-connectx-9-supernic-analysis-1x800g-ethernet-breakthrough), [NVIDIA Ethernet SuperNICs](https://www.nvidia.com/en-us/networking/products/ethernet/supernic/)

### 2.4 PCIe Gen 6 主机接口

BlueField-4 通过 16 通道 PCIe Gen 6 接口连接主机（或在无主机的 CMX 部署中连接存储背板）。PCIe Gen 6 将每通道带宽从 Gen 5 的 32 GT/s 翻倍至 64 GT/s，提供约 128 GB/s 的聚合双向带宽。这对于 CMX 机箱至关重要，在这些场景下 BlueField-4 必须在大量 SSD 间维持高吞吐量的 NVMe 命令提交和完成。

### 2.5 硬件加速引擎

BlueField-4 继承并扩展了前几代产品的硬件加速器组合：

- **加密加速**：800 Gb/s 线速 AES 加解密。覆盖 NVMe SSD 上 KV cache 的静态加密以及 RDMA 传输过程中的动态加密。加密引擎以内联方式工作——数据直接流过而无需存储转发缓冲——因此对数据路径几乎不增加延迟。

- **CRC/数据完整性**：800 Gb/s 线速硬件 CRC 计算。用于 KV cache 数据块的 T10-DIF 式端到端数据完整性保护。写入 NVMe 的每个数据块都受硬件生成的校验和保护；读回的每个数据块在通过 RDMA 发送给请求方 GPU 节点之前都会经过验证。

- **压缩/解压缩**：硬件辅助压缩（记录在 DOCA/DPDK mlx5 压缩驱动文档中），用于减少 KV cache 在闪存上的存储占用。这对于注意力机制的 KV 数据块尤为重要，此类数据通常具有可观的压缩比，能够有效扩展闪存可用容量。

- **DMA 引擎**：高性能 DMA，用于在主机内存、DPU 内存、NVMe 后端和网络引擎之间搬移数据，无需 CPU 介入。

- **正则表达式（RegEx）加速**：从 BlueField-3 延续而来，主要用于深度包检测和安全策略执行，而非存储操作。

这些加速器的关键意义在于：它们使 BlueField-4 能够在 800 Gb/s 全线速下对 KV cache 数据执行加密、完整性校验和压缩，**而无需消耗 Grace CPU 周期**。CPU 算力可完全用于控制平面逻辑、元数据管理、遥测和 DOCA 服务运行。

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [DPDK mlx5 Compress Driver](https://doc.dpdk.org/guides/compressdevs/mlx5.html), [Chiplog: Analysis of BlueField-4](https://www.chiplog.io/p/analysis-of-nvidias-bluefield-4-dpu)

---

## 3. 存储处理器角色

### 3.1 NVMe SSD 管理

在 CMX 部署中，BlueField-4 充当机箱内 SSD 的 NVMe 控制器。它不仅仅是透传主机的 NVMe 命令——而是**拥有** NVMe 命名空间、管理命令队列、处理磨损均衡策略，并在整个驱动器池中编排数据放置。机箱中的主机 CPU（如果存在的话）以及网络上的 GPU 节点永远不会直接与 SSD 交互；它们通过 DOCA Memos API 或 NVMe-oF 与 BlueField-4 交互。

这是与传统 DPU 存储架构的根本性区别。在传统架构中，DPU 充当 virtio-blk 或 NVMe SNAP 仿真层，将远程存储呈现为本地驱动器。在 CMX 中，BlueField-4 就是**存储控制器本身**。它原生运行 NVMe 驱动逻辑，使用 SPDK（Storage Performance Development Kit，存储性能开发套件）块设备后端进行 NVMe 命名空间管理。

BlueField-4 支持标准 NVMe 和 NVMe-oF 传输协议，包括 **NVMe KV（键值）命令集扩展**。NVMe KV 命令集（在 NVMe 规范中定义）在协议层面提供原生键值语义——对可变长度键和值的 put、get、delete 和 list 操作——这与 KV cache 管理面向块的特性天然契合。这不是在块存储之上叠加的应用层抽象，而是 NVMe SSD 和 BlueField-4 NVMe 控制器能够协同利用的协议级能力。

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [NVIDIA BlueField-3 SNAP Documentation](https://docs.nvidia.com/networking/display/bluefield3snap440/introduction)

### 3.2 数据完整性卸载

KV cache 的数据完整性不容忽视——KV 数据块中的一个比特翻转就可能损坏整个推理会话，导致无意义的输出，甚至更糟的是产生细微的错误结果。BlueField-4 通过在数据路径的每个阶段实施硬件加速的完整性校验来应对这一挑战：

- **写路径**：当 KV 数据块通过 RDMA 从 GPU 节点到达时，BlueField-4 的 CRC 引擎在将数据块写入 NVMe 之前计算校验和。校验和与数据一同存储，遵循类似 T10-DIF（数据完整性字段）端到端保护的模型。

- **读路径**：当 KV 数据块从 NVMe 读出并准备发送给 GPU 节点时，CRC 引擎根据数据验证存储的校验和。如果校验失败，该数据块将被标记为损坏，推理框架可以重新计算它而非使用过期或受损的上下文。

- **传输中保护**：BlueField-4 与 GPU 节点之间的 RDMA 传输受 RoCE 传输层自身 CRC 机制的保护，在存储层完整性之上增加了网络层完整性保障。

所有这些操作都在专用硬件中以 800 Gb/s 线速完成。Grace CPU 不参与数据路径上的校验和计算或验证。

### 3.3 静态加密

BlueField-4 的加密加速引擎为所有写入 NVMe SSD 的 KV cache 数据提供 AES-256 加密。在多租户云环境中，这确保了一个租户的 KV cache 无法被另一个租户读取，即使物理驱动器被共享或退役。

加密在写路径上内联执行，解密在读路径上内联执行，均达到 800 Gb/s 速率。密钥管理由运行在 Grace CPU 上的 DOCA 安全服务负责，结合从前几代 BlueField 继承的硬件信任根机制，确保加密密钥不会暴露给主机或网络。

对于习惯使用 SED（自加密驱动器）的存储工程师而言，BlueField-4 的模型有所不同：加密由**存储处理器**执行，而非驱动器本身。这赋予了存储运维人员更大的灵活性——可按租户、按会话或按 KV 命名空间使用不同的加密密钥——而无需依赖底层 NVMe 驱动器的 SED 能力。

### 3.4 KV cache 压缩

大语言模型推理过程中生成的 KV cache 数据块具有可观的可压缩性。注意力机制的键和值张量通常包含冗余模式，尤其是在长上下文会话中，大量上下文窗口在多轮对话间共享的情况下更是如此。

BlueField-4 的硬件压缩引擎能够在将 KV 数据块写入 NVMe 之前进行压缩，并在读取时解压缩。这有效地增加了闪存层的可用容量，而无需配置更大或更多的驱动器。压缩对推理框架完全透明——DOCA Memos 在内部处理这一过程，向消费端呈现未压缩的 KV 数据块。

代价是延迟：压缩和解压缩会增加写路径和读路径的处理时间。对于 KV cache 工作负载——数据是临时性的且访问模式是顺序的（预填充一次，解码阶段读取），容量收益通常超过延迟成本。特别是当替代方案是缓存未命中导致完整上下文重计算时，压缩带来的收益更为显著。

### 3.5 从主机 CPU 和 GPU 卸载

BlueField-4 作为存储处理器的核心价值主张在于卸载。在传统存储架构中，主机 CPU 负责 NVMe 驱动执行、校验和计算、加密、压缩和网络协议处理。在以 GPU 为中心的 AI 集群中，将主机 CPU 周期用于存储操作会直接减少可用于推理编排的算力。

BlueField-4 彻底消除了这一开销。GPU 节点与 CMX 存储的交互通过 RDMA 完成——GPU（或其主机 CPU）发起一个 DOCA Memos API 调用，该调用转化为发往 BlueField-4 节点的 RDMA 传输。此后，所有 NVMe 命令处理、完整性校验、加密和压缩都在 BlueField-4 的硅芯片上完成。GPU 节点的主机 CPU 不参与任何存储 I/O 处理。

NVIDIA 将这一优势量化为：与传统基于 CPU 的存储架构相比，可实现最高 **5 倍的每秒 token 生成速率**和 **4 倍的能效提升**，以及 **2 倍的 RAG（Retrieval-Augmented Generation，检索增强生成）页面摄取速度**。

Sources: [Tom's Hardware: BlueField-4 STX for Agentic AI](https://www.tomshardware.com/tech-industry/nvidia-launches-bluefield-4-stx-storage-architecture-for-agentic-ai), [VentureBeat: BlueField-4 STX Context Memory](https://venturebeat.com/data/nvidia-bluefield-4-stx-adds-a-context-memory-layer-to-storage-to-close-the)

---

## 4. DOCA Memos SDK

### 4.1 定位与目标

DOCA Memos 是推理框架与 CMX 存储层之间的软件接口。它是 DOCA（Data Center Infrastructure on a Chip Architecture，片上数据中心基础设施架构）软件框架中针对 BlueField-4 和 CMX 优化的 SDK。其目标非常明确：**将 KV cache 存储以键值服务的形式暴露出来**，使推理引擎可以直接使用，而无需关心底层的 NVMe 驱动器、RDMA 传输或加密机制。

DOCA Memos 将以太网连接的闪存转化为 Pod 级缓存层。应用保持无状态——通过 API 执行 put 和 get KV 数据块操作——而 CMX 在幕后处理路由、放置、副本、完整性和生命周期管理。

Sources: [NVIDIA CMX Context Memory Storage Platform](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/), [Blocks and Files: NVIDIA Context Memory Extension](https://blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/)

### 4.2 键值 API 接口

DOCA Memos 暴露的键值语义与 NVMe KV 命令集及推理 KV cache 管理的操作模型保持一致。核心操作包括：

- **Put**：将 KV cache 数据块（或一组数据块）存储到 CMX 层，关联一个键——该键通常编码了模型 ID、层索引、序列哈希或会话标识符。put 操作触发完整的写入流水线：完整性校验和生成、可选压缩、加密，以及 NVMe 写入目标 SSD。

- **Get**：按键检索 KV cache 数据块。get 操作执行反向的写入流水线：NVMe 读取、完整性验证、解密、可选解压缩，以及 RDMA 传输到请求方 GPU 节点。如果数据块已被驱逐或过期，get 返回缓存未命中信号，提示推理框架重新计算 KV 上下文。

- **Delete**：显式删除 KV cache 数据块。用于推理会话结束、上下文窗口失效，或框架判定缓存的上下文不再可复用时。

- **List**：枚举匹配模式或前缀的键。供推理框架进行缓存发现——在决定重新计算还是复用上下文之前，确定哪些 KV 数据块已在 CMX 上可用。

这些操作与 BlueField-4 NVMe 控制器支持的 NVMe KV 命令集扩展天然映射，实现了从 API 调用到协议级操作的清晰转换，无需中间的块转换层。

### 4.3 缓存管理：驱逐、TTL 与优先级

KV cache 天生具有临时性。与传统存储数据不同，KV 数据块有自然的过期周期——它们仅在所属的推理会话或对话仍然活跃时才有用。DOCA Memos 通过以下机制实现缓存生命周期管理：

- **TTL（Time-to-Live，生存时间）**：KV 数据块可在写入时分配 TTL。当 TTL 到期时，该数据块变为可驱逐状态。这与 NVIDIA Dynamo 基于 TTL 的缓存固定模型一致，推理路由器可以指定特定对话轮次或会话的 KV 数据块保留时长。

- **LRU 驱逐**：在内存压力下，DOCA Memos 根据 LRU（Least Recently Used，最近最少使用）顺序驱逐数据块。最近未被访问的数据块优先驱逐，释放闪存容量用于新写入。

- **基于优先级的驱逐**：推理框架可以为 KV 数据块分配优先级。系统提示词的 KV cache（跨多个会话共享）可以固定为高优先级，而不太可能被复用的单轮上下文可以分配低优先级。在驱逐压力下，低优先级数据块先于高优先级数据块被驱逐，无论访问时间先后。

- **分层降级**：DOCA Memos 不是直接删除被驱逐的数据块，而是可以将数据块从 GPU HBM（高带宽内存）降级到主机/Grace CPU 内存，再从 CPU 内存降级到 CMX 闪存。这种分层方式与 NIXL 协同编排，意味着从一个层级的"驱逐"就是下一个层级的"摄入"，以更低的性能层级保留上下文以供潜在复用。

这些策略可按命名空间或按租户配置，支持不同推理工作负载具有不同缓存保留需求的多租户部署场景。

Sources: [NVIDIA Developer Blog: KV Cache Reuse in TensorRT-LLM](https://developer.nvidia.com/blog/introducing-new-kv-cache-reuse-optimizations-in-nvidia-tensorrt-llm/), [NVIDIA Developer Blog: Reducing KV Cache Bottlenecks with Dynamo](https://developer.nvidia.com/blog/how-to-reduce-kv-cache-bottlenecks-with-nvidia-dynamo/), [NAND Research: NVIDIA STX and CMX](https://nand-research.com/nvidia-stx-cmx-infrastructure-for-agentic-ai-context-storage/)

### 4.4 多租户

在云和多租户部署中，CMX 必须在网络、存储和 API 的每个层面隔离不同租户的 KV cache。DOCA Memos 通过以下方式支持多租户：

- **命名空间隔离**：每个租户（或推理服务）在专用的 KV 命名空间中运行。命名空间在 BlueField-4 层面强制执行——租户的 API 凭据仅授权访问其命名空间，DOCA 运行时禁止跨命名空间访问。

- **按租户加密密钥**：BlueField-4 的加密引擎支持按命名空间的加密密钥，确保即使物理 NVMe 容量在租户间共享，静态数据也实现了加密隔离。

- **网络层隔离**：BlueField DPU 接入每个租户的 VPN 或网络分区，呈现隔离的虚拟 NIC。DPU 确保一个租户的 KV cache 流量不会被同一物理基础设施上的其他租户观察或拦截。

- **QoS（服务质量）**：DOCA Memos 可以强制执行按租户的带宽和 IOPS 限制，防止嘈杂邻居式的推理工作负载挤占其他租户的 CMX 吞吐量。

这一多租户模型延伸到 Spectrum-X 以太网交换架构，后者提供高级拥塞控制和自适应路由，即使多个租户共享同一 Pod 级 CMX 基础设施，也能保持一致、可预测的 RDMA 性能。

### 4.5 推理框架集成

DOCA Memos 设计用于与 NVIDIA 推理软件栈集成，具体包括：

- **NVIDIA Dynamo**：分布式推理框架，负责编排 KV cache 在各内存层级间的放置和移动。Dynamo 的 KV 数据块管理器使用 DOCA Memos 作为 CMX 存储操作的 API。当 Dynamo 决定将 KV 数据块从 GPU HBM 卸载到闪存时，它通过 DOCA Memos 发起 put 操作。当需要为即将到来的解码阶段预载 KV 数据块时，它发起 get 操作。

- **TensorRT-LLM**：NVIDIA 面向大语言模型的优化推理引擎。TensorRT-LLM 的 KV cache 复用优化——包括前缀缓存和跨请求缓存共享——直接受益于 DOCA Memos。当多个推理请求共享相同的系统提示词时，该提示词的 KV cache 可以在 CMX 上存储一次，由任意 GPU 节点检索，消除冗余重计算。

- **vLLM**：开源推理引擎的 PagedAttention 机制在 GPU 内存中以近零浪费的方式管理 KV cache。DOCA Memos 将这一模型扩展到闪存——当 GPU 内存满载时，vLLM 可以通过 DOCA Memos 将 KV 数据块换出到 CMX，并在需要时换入。

- **NVIDIA NIXL（Inference Transfer Library，推理传输库）**：NIXL 是 DOCA Memos 用于在内存层级间移动 KV 数据块的传输层。它提供统一的 API，无论传输是通过 NVLink、InfiniBand、RoCE 还是以太网，并支持异步操作，使 GPU 计算不会因 KV 传输而阻塞。

集成模型是分层的：推理框架调用 Dynamo 的 KV 管理器，后者调用 DOCA Memos API，再转化为 BlueField-4 上的 NIXL 传输和 NVMe 操作。每一层抽象了下层的复杂性。

Sources: [NVIDIA Developer Blog: Dynamo Low-Latency Inference](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/), [NVIDIA Developer Blog: KV Cache Bottlenecks with Dynamo](https://developer.nvidia.com/blog/how-to-reduce-kv-cache-bottlenecks-with-nvidia-dynamo/), [Blocks and Files: NVIDIA KV Cache Offload to NVMe](https://blocksandfiles.com/2026/01/06/nvidia-standardizes-gpu-cluster-kv-cache-offload-to-nvme-ssds/)

---

## 5. 软件栈架构

### 5.1 BlueField-4 上的 DOCA 运行时

BlueField-4 在其 64 个 Arm Neoverse V2 核心上运行完整的 Linux 操作环境，DOCA 运行时作为主要软件平台。DOCA 运行时包括：

- **操作系统**：针对 Arm 架构优化的标准 Linux 发行版（通常基于 Ubuntu），直接运行在 BlueField-4 内部的 Grace CPU 核心上。

- **DOCA SDK**：行业标准驱动和库，包括 DPDK（Data Plane Development Kit，数据平面开发套件）用于高性能包处理，SPDK（Storage Performance Development Kit，存储性能开发套件）用于 NVMe 设备管理，以及 P4 可编程性用于自定义包处理流水线。

- **DOCA 微服务**：运行在 BlueField-4 上的容器化服务，提供网络、存储、安全和遥测功能。在 CMX 部署中，关键微服务是 DOCA Memos 服务，它实现 KV cache 管理逻辑并向网络暴露键值 API。

- **容器编排**：DOCA Platform（作为开源项目在 GitHub 上提供）管理数据中心中 BlueField DPU 的配置和服务编排。它支持大规模部署、更新和监控 DOCA 微服务——这在 CMX 部署可能包含多个机箱中数百颗 BlueField-4 处理器的场景下至关重要。

BlueField-4 上的 DOCA 运行时完全自包含。它从板载 512 GB SSD 启动，运行自己的网络栈，独立于任何主机操作系统管理机箱中的 NVMe SSD。在无主机 CMX 机箱（没有主机 CPU）中，BlueField-4 **就是**机箱控制器。

Sources: [NVIDIA DOCA Software Framework](https://developer.nvidia.com/networking/doca), [NVIDIA DOCA Documentation](https://docs.nvidia.com/doca/sdk/index.html), [GitHub: NVIDIA DOCA Platform](https://github.com/NVIDIA/doca-platform)

### 5.2 与 GPU 节点的 RDMA 通信

GPU 计算节点与 CMX 存储节点之间的数据路径基于 RDMA 构建。一次 KV cache 读操作的流程如下：

1. 推理框架（例如运行在 GPU 节点上的 Dynamo）判定需要一个存储在 CMX 上的 KV 数据块。
2. 框架调用 DOCA Memos 客户端库，传入目标 KV 数据块的键。
3. DOCA Memos 客户端调用 NIXL 发起从 CMX 节点的 RDMA 读取。
4. 在 CMX 节点上，BlueField-4 的 ConnectX-9 引擎接收 RDMA 请求并将其路由到 DOCA Memos 服务。
5. DOCA Memos 服务在元数据索引中查找键，定位 NVMe 位置，并通过 SPDK 发起 NVMe 读取。
6. 数据从 NVMe SSD 流经 BlueField-4 的完整性验证引擎（CRC 校验）、解密引擎（AES-256）和可选的解压缩引擎。
7. 处理后的数据被放入 RDMA 发送缓冲区，通过 Spectrum-X 以太网交换架构传回 GPU 节点。
8. 在 GPU 节点上，NIXL 通过 GPUDirect RDMA 将 KV 数据块直接送入 GPU HBM（或主机内存）。

整个操作在数据路径上绕过了 CMX 节点的 Grace CPU——第 5 到第 7 步由硬件加速器和 DMA 引擎处理。Grace CPU 仅参与元数据查找（第 5 步）和控制平面编排。

对于写操作，流程相反：GPU 节点通过 RDMA 将 KV 数据块推送到 CMX，BlueField-4 在硬件中执行压缩、加密和 CRC 生成，然后将结果写入 NVMe。

### 5.3 NVMe 驱动交互

BlueField-4 通过 SPDK 管理 NVMe SSD，完全绕过内核的 NVMe 驱动。SPDK 提供用户空间 NVMe 驱动，直接轮询完成队列，消除中断开销和上下文切换延迟。这对延迟敏感的 KV cache 工作负载至关重要——单次 get 操作必须在微秒级完成，以避免阻塞推理流水线。

CMX 中的 NVMe 交互模型具有以下几个显著特征：

- **NVMe KV 命令集**：在底层 SSD 支持的情况下，BlueField-4 可以原生使用 NVMe KV 命令集（KV Retrieve、KV Store、KV Delete、KV List），将 DOCA Memos API 调用直接映射到 NVMe KV 命令。这消除了基于块的 NVMe 所需的键到 LBA（逻辑块地址）转换层。

- **NVMe-oF 目标**：BlueField-4 还可以通过 NVMe-oF（Fabrics）暴露 NVMe 命名空间，使远程 GPU 节点能够通过标准 NVMe-oF 发起端访问 SSD。这为偏好块级访问而非键值访问的框架提供了备选路径。

- **SNAP 仿真**：为了向后兼容期望看到本地 NVMe 设备的主机，BlueField-4 支持 SNAP（Software-defined Network Accelerated Processing，软件定义网络加速处理），在 PCIe 总线上仿真本地 NVMe 控制器。在 CMX 中，由于主要访问模型是基于网络的，这一功能较少使用，但在混合部署中仍然可用。

- **驱动器健康管理**：BlueField-4 监控 SSD 健康遥测（SMART 属性、磨损指标、错误率）并可在驱动器故障前主动迁移数据。64 核 Grace CPU 提供充裕的算力来运行预测性维护算法，不影响数据路径性能。

### 5.4 G3.5 存储层

CMX 在 AI 数据中心内存层次结构中建立了 NVIDIA 所称的"G3.5"存储层：

| 层级 | 介质 | 范围 | 延迟 | 容量 |
|---|---|---|---|---|
| G1 | GPU HBM | 单 GPU | ~纳秒级 | 数十 GB |
| G2 | 主机/Grace CPU DRAM | 单节点 | ~数百纳秒 | 数百 GB |
| G3 | 本地 NVMe | 单节点 | ~微秒级 | TB 级 |
| **G3.5** | **CMX（Pod 级闪存）** | **Pod 范围** | **~数十微秒** | **PB 级** |
| G4 | 共享企业存储 | 集群范围 | ~毫秒级 | EB 级 |

G3.5 层定位独特：它兼具共享存储的容量与持久性以及本地闪存的低延迟和高带宽，通过 RDMA 可被 Pod 内所有节点访问。对于 KV cache——临时性、高带宽，且从跨推理会话共享中获益巨大——G3.5 填补了本地 SSD 和企业存储阵列都无法高效满足的空白。

DOCA Memos 和 NIXL 协同管理 KV 数据块在这些层级间的移动。当 GPU HBM 满载时，KV 数据块降级到 G2（主机 DRAM），再到 G3.5（CMX 闪存）。当即将到来的解码阶段需要某个 KV 数据块时，它会提前从 G3.5 预载到 G2 或 G1，通过预取隐藏闪存访问延迟。

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [NAND Research: Improving Inference with NVIDIA ICMS](https://nand-research.com/research-note-improving-inference-nvidias-inference-context-memory-storage-platform/)

### 5.5 监控与遥测

BlueField-4 的 64 核 Grace CPU 为与数据路径工作负载并行运行的遥测和监控服务提供了充裕的算力余量。DOCA 运行时支持：

- **驱动器遥测**：实时采集所有被管理 SSD 的 NVMe SMART 数据、磨损均衡统计、错误率和温度读数。这些数据馈入预测性维护模型，可以在驱动器故障前主动重新分布数据。

- **KV cache 分析**：缓存命中率、驱逐率、put/get 延迟分布、按租户利用率和压缩比等指标。这些指标对于调优驱逐策略、TTL 值和容量规划至关重要。

- **网络遥测**：Spectrum-X 交换架构指标，包括 RDMA 吞吐量、延迟直方图、拥塞事件和自适应路由决策。BlueField-4 可以关联存储和网络遥测数据以识别瓶颈——例如检测 KV cache 未命中延迟是由网络拥塞而非 NVMe 访问时间主导。

- **机箱管理**：CMX 机箱的功耗、温度状态、风扇转速和硬件健康状况。BlueField-4 在无主机机箱中充当基板管理控制器，提供带外管理接口。

遥测数据可以通过标准协议（Prometheus、gRPC、SNMP）导出到数据中心监控平台，使 CMX 存储基础设施能够在统一的运维仪表板中与传统存储阵列和网络设备一同管理。

Sources: [VAST Data: Advancing State of the Art with BlueField-4](https://www.vastdata.com/blog/advancing-the-state-of-the-art-with-nvidia-blue-field-4), [SiliconANGLE: Convergence of Context with BlueField-4 STX](https://siliconangle.com/2026/03/20/convergence-context-nvidias-bluefield-4-stx-marries-network-storage-admin/)

---

## 6. 核心要点

**BlueField-4 不是附带 CPU 边车的网卡——它是一颗完整的存储处理器。** 64 核 Grace CPU、128 GB LPDDR5 和 800 Gb/s ConnectX-9 引擎使其能够独立管理 NVMe 机箱、运行容器化存储服务，并在硬件中以线速处理所有数据路径操作（加密、完整性、压缩）。存储工程师应将其视为嵌入式存储控制器，而非网络适配器。

**DOCA Memos 将 NVMe 抽象为键值服务。** 传统存储栈——块设备、文件系统、卷管理器——被一个专为推理上下文优化的 KV API 所取代。这对存储运维而言是一次范式转变：配置、监控和容量规划以 KV 命名空间、驱逐策略和 TTL 来衡量，而非 LUN、RAID 组和文件系统区段。

**G3.5 层改变了容量规划方式。** CMX 在存储层次结构中创建了一个此前不存在的新层级——具有 RDMA 访问能力的 Pod 级闪存。存储工程师现在必须在规划本地 SSD（G3）和共享存储（G4）的同时考虑这一层级，需要评估 KV 数据块大小、会话持续时间、缓存命中率和跨 Pod 流量模式等因素。

**硬件卸载是性能的关键使能因素。** 相比基于 CPU 的存储方案实现的 5 倍吞吐量提升，并非来自更快的 SSD——而是来自消除数据路径上的 CPU 开销。BlueField-4 在硬件中处理的每一个存储操作（校验和、加密、压缩、NVMe 命令），都意味着 GPU 节点的主机 CPU 不必消耗相应的计算周期，直接转化为更高的推理吞吐量。

**多租户是原生内建的，而非事后拼凑的。** 按租户加密密钥、命名空间隔离、网络分段和 QoS 执行都是 BlueField-4 和 DOCA Memos 架构的原生能力。这是云服务商在多个推理客户间部署共享 CMX 基础设施的必要条件。

**生态系统是护城河。** 在 GTC 2026 上，NVIDIA 宣布十家主要存储厂商采用 BlueField-4 STX。DOCA Memos 与 Dynamo、TensorRT-LLM、vLLM 和 NIXL 原生集成。对于评估 CMX 的存储工程师而言，问题不仅在于硬件是否满足需求——更在于他们部署的推理软件栈是否会假定 CMX 的存在并据此优化。

Sources: [NVIDIA Newsroom: BlueField-4 STX Broad Industry Adoption](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption), [SDxCentral: BlueField-4 First Look](https://www.sdxcentral.com/analysis/nvidias-bluefield-4-a-first-look-at-the-dpu-built-to-run-ai-factories/)
