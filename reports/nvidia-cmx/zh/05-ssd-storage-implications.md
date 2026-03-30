# NVIDIA CMX 对 SSD 与存储的影响

**面向存储工程师的深度技术分析：评估 AI 原生推理基础设施中的闪存需求**

> **摘要：** NVIDIA 的 Context Memory Storage（CMX）平台于 GTC 2026 发布，是 BlueField-4 STX 架构的首个落地实现，开创了一个全新的 SSD 工作负载类别：AI 原生上下文存储。CMX 将 GPU 内存层级扩展至闪存支撑的 KV cache 层，对 NVMe SSD 提出了一套与传统数据中心存储工作负载截然不同的独特需求。本报告从容量、耐久性、延迟和接口四个维度分析这些需求，评估 QLC（四层单元）NAND 对 CMX 工作负载的适配性，梳理 SSD 供应商生态，并预测 2028 年前的市场影响。本报告面向规划 CMX 闪存采购与部署的存储工程师和基础设施架构师。

---

## 1. 引言：CMX 创造了全新的 SSD 工作负载类别

过去十年间，企业级 SSD 工作负载已形成清晰的分类谱系：写入密集型（数据库、日志）、混合型（虚拟化、通用计算）和读取密集型（CDN、分析、模型服务）。NVIDIA 的 CMX 平台引入的工作负载不属于上述任何类别，其本质可概括为**短暂上下文存储**——高容量、延迟敏感、写入高频换页，但总体以读取为主。

其技术驱动力是 KV cache。大语言模型推理会为对话上下文窗口中的每个 token 生成键值对。随着上下文窗口扩展到百万级 token，以及 Agentic AI（智能代理）系统维护长周期多轮会话，单个会话的 KV cache 占用可达数 GB。当并发会话的聚合 KV cache 超出 GPU HBM（高带宽内存）容量时，系统必须将上下文溢出到更低层级——先到主机 DRAM，再到本地 NVMe SSD，最后通过 CMX 到达共享的 Pod 级闪存([NVIDIA CMX Platform](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/))。

NVIDIA CMX 正是提供这一 Pod 级闪存层的平台。CMX 基于 BlueField-4 DPU（数据处理单元）和 Spectrum-X 以太网，将 NVMe SSD 管理为共享的、RDMA 加速的上下文内存池，Pod 内任何 GPU 均可访问。NVIDIA 宣称与传统存储方案相比，CMX 可实现高达 5 倍的 tokens/秒吞吐提升和 5 倍的能效提升([NVIDIA BlueField-4 STX Announcement](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption))。软件层 NVIDIA Dynamo 负责 KV 感知的路由与放置，使闪存层对推理工作负载完全透明([NVIDIA Dynamo KVBM Design](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design))。

对存储工程师而言，结论很明确：每个 Vera Rubin 推理 Pod 都需要一个专用闪存层，而填充该闪存层的 SSD 面临的工作负载特征与现有任何资格认证手册中的描述都截然不同。

---

## 2. CMX SSD 需求

### 2.1 容量：单节点 TB 级，单 Pod PB 级

CMX 在 Pod 级别运行。Vera Rubin NVL72 机柜集成 72 颗 GPU，每颗 GPU 配备 HBM4 本地内存。当 KV cache 溢出超过 HBM 和主机 DRAM 后，CMX 闪存层必须承载潜在数千并发推理会话的上下文数据。

**单会话 KV cache 容量估算。** 以一个 70B 参数的大语言模型、1M token 上下文窗口为例，每个会话的 KV cache 大约消耗 2-4 GB（具体取决于量化方案和模型架构）。405B 参数模型在相当上下文长度下可能需要 8-16 GB/会话。在每 Pod 1,000 并发会话的情况下，聚合 KV cache 占用达 2-16 TB 的活跃上下文数据——这还不包括已完成但可能被复用的会话的留存上下文。

**Pod 级容量。** NVIDIA 将 CMX 描述为"每个 GPU Pod 提供 PB 级共享容量"([NVIDIA Developer Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/))。实际上，来自 Supermicro、AIC 等合作伙伴的早期 CMX 参考设计采用高密度 NVMe 硬盘填充存储节点。一个配备 24 块 25.6 TB SSD 的 CMX 存储节点可提供约 614 TB 的原始闪存容量。一个 Pod 配置 4-8 个此类存储节点即可达到 2.5-5 PB。

**对 SSD 选型的意义。** 单盘容量至关重要。15.36 TB 的硬盘虽可使用，但需要更多插槽，增加了布线和功耗开销。CMX 的最佳容量区间可能在 25.6-61.44 TB/盘，而 122 TB+ 的硬盘在量产成熟后将更具吸引力。

### 2.2 读取延迟：硬盘层面 Sub-100 微秒

CMX 的价值主张取决于将缓存的 KV 块回传给 GPU 的速度快于重新计算的时间。延迟预算非常紧凑：

- **GPU HBM 访问：** ~1 us
- **主机 DRAM 访问：** ~0.1-0.5 us（本地），~1-5 us（CXL 挂载）
- **CMX 闪存层目标：** 硬盘端 10-80 us，加上网络传输延迟
- **重算代价：** 毫秒到数百毫秒不等，取决于序列长度

为使 CMX 闪存层发挥作用，端到端延迟（硬盘 + BlueField-4 DPU + RDMA 网络 + GPU 接收）必须远低于重算代价。RDMA 网络传输增加 5-15 us，DPU 处理开销为数微秒，因此 SSD 本身必须**稳定在 sub-80 us 范围内**交付读取结果，不仅是中位数，还包括 P99（第 99 百分位）延迟。

对于该工作负载特征中以 33 MB KV cache 块为主的顺序读取，当前企业级 NVMe SSD 可以满足这一要求。但 QoS（服务质量）一致性比峰值吞吐更重要。来自垃圾回收、SLC 到 QLC 折叠或读取重试操作的尾延迟突增可能导致 GPU 停顿。部署在 CMX 中的硬盘必须在持续混合工作负载下表现出确定性的延迟行为。

### 2.3 写入耐久性：短暂高换页上下文

KV cache 本质上是短暂的。上下文块在会话创建或扩展时写入，在会话结束或缓存承压时被驱逐。写入模式为：

- **大块顺序写入：** KV 块通常以连续的 2-33 MB 块写入（因框架和量化方案而异）。
- **活跃数据高频换页：** 持续数分钟到数小时的会话生成的上下文写入一次，可能被读取多次，然后丢弃。
- **频率门控的磁盘卸载：** NVIDIA Dynamo 的 KVBM（KV Block Manager，KV 块管理器）默认实现基于频率的磁盘卸载过滤，仅将复用频率 ≥ 2 的 KV 块卸载到 SSD。这是一种显式的耐久性保护机制([NVIDIA Dynamo KVBM Documentation](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design))。

**写入量估算。** 假设一个 CMX 节点拥有 614 TB 闪存，服务于一个包含 1,000 并发会话的 Pod，每个会话生成 4 GB 的 KV cache，平均会话生命周期为 30 分钟。若 50% 的会话满足磁盘卸载条件（频率 ≥ 2），则产生：

- 500 会话 × 4 GB = 2 TB 新写入/30 分钟周期
- 全节点每天 96 TB 写入量
- 分布在 24 块 25.6 TB 硬盘上 = 每盘每天 4 TB
- DWPD（每日全盘写入次数）= 4 / 25.6 = ~0.16 DWPD

这是保守估算。在激进的 Agentic 工作负载下——快速创建会话、更长上下文——写入量可能达到 0.5-1.0 DWPD/盘。写入模式以顺序为主（有利于 NAND 耐久性），但换页是持续进行的。

**DWPD 需求范围：** 根据工作负载强度，设计范围为 0.3-1.0 DWPD。这恰好处于读取密集型 QLC（0.3-0.58 DWPD）和主流 TLC（1-3 DWPD）的分界线上。

### 2.4 访问模式：顺序写入、混合读取

对 KV cache 设备级 I/O 模式的研究表明，约 78% 的访问为顺序读取，每个推理服务对其分配的 KV cache 块发起专用顺序读取([An I/O Characterizing Study of Offloading LLM Models and KV Caches to NVMe SSD](https://atlarge-research.com/pdfs/2025-cheops-llm.pdf))。Linux 块层将每个 33 MB 的 KV cache 块分割为以 2.8 MB 和 6.1 MB 为主的子请求，由于多个并发服务的请求流在时间上重叠，NVMe 设备会接收到多个并发的顺序读取序列。

这意味着 SSD 检测和维持交错流中顺序访问模式的能力至关重要。具备积极预读和多流感知能力的硬盘将优于那些纯粹为随机 4K IOPS 优化的硬盘。

### 2.5 接口：当下 PCIe Gen5，Gen6 即将到来

Vera Rubin 平台支持 PCIe Gen6 和 CXL 3.1([NVIDIA Vera Rubin Platform](https://developer.nvidia.com/blog/inside-the-nvidia-rubin-platform-six-new-chips-one-ai-supercomputer/))。2026 年下半年的早期 CMX 部署将使用 PCIe Gen5 NVMe SSD，因为这是目前经过资格认证且可大规模供应的产品。Micron 9650——全球首款 PCIe Gen6 数据中心 SSD，自 2026 年 2 月量产——提供 28 GB/s 顺序读取性能，明确定位于 BlueField-4 STX 工作负载([Micron 9650 for Vera Rubin](https://investors.micron.com/news-releases/news-release-details/micron-high-volume-production-hbm4-designed-nvidia-vera-rubin))。

对于需要向 Pod 提供 37+ GB/s 聚合吞吐的 CMX 存储节点（如在多并发用户测试环境中观察到的），PCIe Gen5 x4（~14 GB/s/盘）在足够数量并行硬盘配合下可以满足需求。PCIe Gen6 x4（~28 GB/s/盘）在相同聚合带宽下将硬盘数量减半，提升密度和能效。

**规格尺寸。** CMX 存储节点采用 EDSFF 规格。KIOXIA CM9 采用 E3.S 规格、25.6 TB 容量；Micron 9650 采用 E1.S 规格。E3.S 凭借其散热余量和容量密度，是高容量 CMX 节点的首选规格，而 E1.S 在带宽优化配置中仍有其适用场景。如[技术路线图](../../en/05-technology-roadmap-2025-2030.md)所述，预计到 2029 年 E3.S 市场份额将超过 35%，到 2026 年已在超大规模部署中占据主导地位。

---

## 3. QLC SSD 对 CMX 的适配性

### 3.1 读取密集的检索与 QLC 优势高度契合

CMX 最根本的 I/O 特征——写入一次（或少数几次）、多次读取——与 QLC NAND 设计面向的工作负载特征高度吻合。如 [QLC 技术基础](../../en/01-qlc-technology-fundamentals.md)中所述，QLC 读取延迟（tR）为 75-120 us，仅略高于 TLC 的 50-75 us。对于 KV cache 检索中占主导的大块顺序读取（2.8-33 MB 块），单页读取延迟差异在数千页的聚合吞吐下被摊薄，总体吞吐量几乎相当。

QLC 相比 TLC 的读取性能优势是经济层面的：同等投入下，QLC 可提供约 33% 更多的容量。在需要 PB 级闪存的 CMX Pod 中，这意味着每个 Pod 数十万美元的成本节省。

### 3.2 短暂 KV Cache 的耐久性分析

耐久性问题需要细致分析。当前企业级 QLC SSD 的额定 DWPD 为 0.3-0.58（例如 Solidigm D5-P5336 在 5 年期限内为 0.58 DWPD，详见 [QLC 技术基础 4.4 节](../../en/01-qlc-technology-fundamentals.md)）。第 2.3 节的分析估算 CMX 写入量为 0.16-1.0 DWPD，具体取决于工作负载强度。

**QLC 适用的场景：** 对于上下文换页适中的推理工作负载——标准聊天机器人服务、文档问答、摘要生成——预估的 0.16-0.3 DWPD 完全在 QLC 的耐久性范围内。Dynamo 的频率门控卸载过滤（仅持久化复用 2 次以上的块）进一步减少了闪存层的写入量，起到软件级耐久性保护的作用。

**需要 TLC 的场景：** 激进的 Agentic 工作负载——多代理编排伴随快速会话创建/销毁、思维链推理伴随频繁上下文更新——可将写入量推至 0.5-1.0 DWPD。在高端场景下，这超出了 QLC 的耐久性额定值。额定 1-3 DWPD 的 TLC 硬盘为此类场景提供了充足的余量。

KIOXIA CM9 是 KIOXIA 专门为 CMX KV cache 工作负载开发的 TLC 硬盘，在 E3.S 规格下提供 25.6 TB 容量和 3 DWPD 耐久性——这一选择明确反映了最坏情况下 CMX 写入换页超出 QLC 极限的现实([KIOXIA CM9 for CMX](https://www.businesswire.com/news/home/20260316056420/en/KIOXIA-Announces-New-SSD-Model-Optimized-for-AI-GPU-Initiated-Workloads))。同样，Samsung 面向 CMX 平台设计的 TLC SSD PM1753 提供 14.5 GB/s 顺序吞吐，其写入耐久性针对高换页 AI 模式进行了专项优化([Samsung PM1753 for CMX](https://esaitech.com/blogs/insights/samsung-pm1753-ssd-ai-and-data-center-storage))。

### 3.3 SLC 缓存的相关性

如 [QLC 技术基础第 6 节](../../en/01-qlc-technology-fundamentals.md)所述，SLC 缓存对 CMX 尤为重要。KV cache 的写入具有突发性：大量新会话或上下文扩展会产生写入浪涌，必须被快速吸收。SLC 缓存以单比特/单元模式在 NAND 原生写入速度下吸收这些突发写入，然后在空闲时段折叠到 QLC。

对于 CMX 中的 QLC，两个特性至关重要：

1. **动态 SLC 缓存大小调整。** 由于 CMX 硬盘可能不会填满（KV cache 是短暂的，持续循环回收），大量未使用的 QLC 块可作为动态 SLC 缓存。一块使用率为 60% 的 61.44 TB QLC 硬盘可维持数百 GB 的 SLC 缓存，从容吸收写入突发。

2. **折叠延迟影响。** SLC 到 QLC 的折叠是一项后台操作，会与 NAND 带宽竞争。折叠期间读取延迟可能出现突增。对于 CMX 延迟敏感的 KV 检索，这是一个隐患。配备智能折叠调度器、能在活跃读取期间延迟折叠操作的硬盘在 CMX 部署中将表现更优。

SanDisk 的 UltraQLC 平台采用 Direct Write QLC（直写 QLC）技术完全绕过 SLC 缓存([SanDisk UltraQLC at FMS 2025](https://www.sandisk.com/company/newsroom/press-releases/2025/2025-08-05-sandisk-showcases-ultraqlc-technology-platform-with-milestone-enterprise-ssd-capacity-at-fms-2025))，可通过消除折叠引发的延迟突增为 CMX 带来优势，但代价是原始写入吞吐量的下降。

### 3.4 容量/成本优势

QLC 在 CMX 大规模部署中的经济性极具竞争力：

| 硬盘 | 技术 | 容量 | 约 $/GB | 614 TB 节点成本 |
|-------|-----------|----------|--------------|---------------------|
| Solidigm D5-P5336 | QLC Gen4 | 122.88 TB | ~$0.10 | ~$61,400（5 块） |
| KIOXIA CM9 | TLC Gen5 | 25.6 TB | ~$0.18-0.22 | ~$110,000-$135,000（24 块） |
| Micron 9650 | TLC Gen6 | 30.72 TB | ~$0.20-0.25 | ~$123,000-$154,000（20 块） |

QLC 可实现闪存层约 50-60% 的成本降低。对于配备 4-8 个 CMX 存储节点的 Pod，仅 SSD 采购一项，QLC 即可节省 $200,000-$750,000。关键问题在于耐久性预算和延迟 QoS 是否能满足特定工作负载的需求。

### 3.5 实践建议：CMX 闪存分层

对于大多数部署场景，最优的 CMX 闪存策略是在 CMX 层内部实施分层：

- **热上下文层（TLC）：** 高耐久性、低延迟的 TLC 硬盘（KIOXIA CM9、Samsung PM1753）承载复用频率最高、换页最频繁的活跃 KV cache。容量规划为 CMX 总容量的 20-30%。
- **温上下文层（QLC）：** 大容量 QLC 硬盘（Solidigm D5-P5336、Micron 6600 ION）承载访问频率较低但必须保持可用的多轮会话留存上下文。容量规划为 CMX 总容量的 70-80%。

NVIDIA Dynamo 的 KV Block Manager 已实现了包括热块提升和冷块降级在内的分层策略([NVIDIA Dynamo KVBM](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design))，在软件层面已具备分层就绪能力。

---

## 4. STX 平台：更广泛的存储架构

### 4.1 STX 架构与定位

STX（Storage Technology eXtension）是 NVIDIA 面向 AI 原生存储的模块化参考架构，基于 BlueField-4 DPU 和 Vera Rubin 平台构建。STX 于 GTC 2026 发布，其范围比 CMX 更广——它定义了面向整个 AI 生命周期的加速存储基础设施应如何构建([NVIDIA STX Architecture](https://www.nvidia.com/en-us/data-center/ai-storage/stx/))。

STX 依托三大核心组件：

- **BlueField-4 DPU：** 面向存储优化的 DPU，负责管理 NVMe SSD、运行存储服务，并卸载数据完整性、加密和压缩任务。BlueField-4 替代了存储 I/O 路径中的传统 CPU，提供更高的每瓦吞吐。
- **ConnectX-9 SuperNIC：** 在计算节点与存储节点之间提供高带宽 RDMA 连接。
- **Spectrum-X 以太网：** 网络交换结构，具备高级拥塞控制、自适应路由和无损 RoCE 功能，确保确定性延迟。

### 4.2 与 CMX 的区别

CMX 是 STX 架构的首个机柜级实现，专门针对 KV cache 上下文内存进行优化。STX 则是通用框架：

| 维度 | CMX | STX（通用） |
|-----------|-----|---------------|
| 主要工作负载 | 推理用 KV cache | 训练数据、检查点、模型仓库、RAG |
| 数据持久性 | 短暂性（小时-天） | 持久性（周-年） |
| 访问模式 | 大块顺序读写、RDMA | 混合随机 + 顺序 |
| SSD 优先级 | 延迟 QoS、容量 | 吞吐、耐久性 |
| 典型 SSD | TLC/QLC NVMe, 25-122 TB | TLC NVMe, 3.84-30.72 TB |

### 4.3 CMX 之外的 STX 使用场景

基于 STX 的存储节点在 AI 数据中心中承担多种角色：

- **训练数据暂存：** 向训练流水线提供打乱后的数据集批次。需要持续的顺序读取带宽。大容量 QLC SSD 非常适合。
- **检查点存储：** 训练过程中定期保存的模型状态快照。数百 GB 到 TB 级的突发写入。需要中等耐久性（1-3 DWPD）。TLC 更优。
- **模型仓库：** 存储和分发部署用的模型权重。读取为主，容量敏感。QLC 是理想选择。
- **RAG 向量数据库：** 检索增强生成需要对嵌入索引的低延迟访问。混合读写，侧重随机读取 IOPS。需高 IOPS 的 TLC。

### 4.4 行业采纳

基于 STX 的平台预计将在 2026 年下半年由合作伙伴推出。已确认的存储系统设计合作伙伴包括 DDN、Dell Technologies、HPE、IBM、NetApp 和 VAST Data，制造合作伙伴包括 Supermicro、AIC 和 Quanta Cloud Technology。Supermicro 已率先展示了结合 NVIDIA Vera CPU 和 BlueField-4 的 CMX 存储服务器原型([Supermicro STX Server](https://ir.supermicro.com/news/news-details/2026/Supermicro-Among-First-to-Unveil-NVIDIA-BlueField-4-STX-Storage-Server-to-Improve-AI-Inference-Performance/default.aspx))。

---

## 5. SSD 供应商就绪状态

### 5.1 CMX 就绪产品

以下 SSD 产品已在 GTC 2026 上明确展示、发布或确认与 CMX/STX 兼容：

**KIOXIA**
- **CM9 系列（TLC, PCIe 5.0, E3.S, 25.6 TB, 3 DWPD）：** 专为 CMX KV cache 打造。14.8 GB/s 读取，11 GB/s 写入，3.4M IOPS。2026 年第三季度提供样品([KIOXIA CM9 Announcement](https://www.businesswire.com/news/home/20260316056420/en/KIOXIA-Announces-New-SSD-Model-Optimized-for-AI-GPU-Initiated-Workloads))。
- **GP 系列（XL-FLASH SLC/MLC, E1.S/E3.S, 800 GB SLC + 1,600 GB MLC）：** 采用存储级内存的超低延迟选项，面向最热的 KV cache 数据。支持 512 字节粒度访问。预计 2026 年底提供评估样品([KIOXIA GP Series](https://www.servethehome.com/kioxia-gp-series-and-cm9-launched-for-the-era-of-agentic-ai-storage/))。

**Micron**
- **9650（TLC, PCIe Gen6, E1.S, 最大 30.72 TB）：** 全球首款 Gen6 SSD，28 GB/s 读取，5.5M 随机读取 IOPS。自 2026 年 2 月量产。明确针对 BlueField-4 STX 优化([Micron 9650](https://www.micron.com/products/storage/ssd/data-center-ssd/9650-ssd))。
- **6600 ION（QLC, PCIe Gen5, E3.S/E3.L, 30.72-245 TB）：** 面向温上下文和模型存储层的大容量选项。122 TB 和 245 TB 版本正在超大规模客户处进行资格认证([Micron 6600 ION](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud))。

**Samsung**
- **PM1753（TLC, PCIe Gen5, E3.S）：** 已确认为 CMX 平台供货。14.5 GB/s 顺序吞吐，3.3M 随机读取 IOPS。低功耗设计，针对推理工作负载优化([Samsung PM1753 for Vera Rubin](https://esaitech.com/blogs/insights/samsung-pm1753-ssd-ai-and-data-center-storage))。

**Solidigm**
- **D5-P5336（QLC, PCIe Gen4, 2.5"/E3.S, 最大 122.88 TB）：** 在 GTC 2026 上于 AIC 的 CMX 对齐 F2026 平台中展示。0.58 DWPD 耐久性。当前出货的最大容量 PCIe SSD([Solidigm D5-P5336](https://www.solidigm.com/products/data-center/d5/p5336.html))。
- **D5-P5436（QLC, PCIe Gen5, 下一代）：** 基于 SK Hynix 321 层 QLC。预计容量突破 122 TB 并具备 Gen5 带宽。

### 5.2 PCIe Gen6 SSD 供应情况

如[技术路线图 6.2 节](../../en/05-technology-roadmap-2025-2030.md)所述，PCIe Gen6 采用 PAM-4 信号编码将每通道带宽翻倍至 64 GT/s。截至 2026 年 3 月，Micron 9650 是唯一量产的 Gen6 SSD。Samsung 和 KIOXIA 尚未发布 Gen6 SSD 产品，但两者均已启动 Gen6 PHY 开发。Solidigm 下一款预期产品为基于 Gen5 的 D5-P5436；Gen6 版本可能在 2027-2028 年间推出。

对于 2026 年下半年启动的 CMX 部署，实际情况是多厂商的 PCIe Gen5 SSD 可供选择，Micron 9650 作为唯一的 Gen6 选项。随着 2027 年 Gen6 产品增多，Gen6 的带宽优势（单盘 2 倍）将在 CMX Pod 规模扩大、单节点硬盘数量减少时变得更加重要。

### 5.3 各供应商 CMX 就绪状态汇总

| 供应商 | CMX 产品 | NAND 类型 | 接口 | 容量 | 耐久性 | 供货时间 |
|--------|-------------|-----------|-----------|----------|-----------|-------------|
| KIOXIA | CM9 | TLC (BiCS8) | Gen5 | 25.6 TB | 3 DWPD | 2026 Q3（样品） |
| KIOXIA | GP Series | SLC/MLC (XL-FLASH) | Gen5 | 2.4 TB | 高 | 2026 年底（评估） |
| Micron | 9650 | TLC (G9) | Gen6 | 最大 30.72 TB | 1 DWPD | 已出货 |
| Micron | 6600 ION | QLC (G9) | Gen5 | 30.72-245 TB | 0.58 DWPD | 出货/认证中 |
| Samsung | PM1753 | TLC (V8) | Gen5 | 待定 | 待定 | 2026 H2 |
| Solidigm | D5-P5336 | QLC (192L) | Gen4 | 最大 122 TB | 0.58 DWPD | 已出货 |
| Solidigm | D5-P5436 | QLC (321L) | Gen5 | >122 TB | 待定 | 2026-2027 |

交叉参考[竞争格局](../../en/02-competitive-landscape.md)：五大 NAND 制造商（Samsung、SK Hynix/Solidigm、Micron、KIOXIA/SanDisk、YMTC）均有与 CMX 相关的产品或路线图，不过 YMTC 受美国出口管制限制，仅参与中国国内市场。

---

## 6. 对 SSD 市场的影响

### 6.1 全新的需求驱动力

CMX 代表了首个 SSD 不是为传统企业存储而设计，而是作为 GPU 内存扩展的存储平台。这改变了采购对话的方式：

- **采购方**是 GPU 基础设施团队，而非存储团队。SSD 选型标准从 $/IOPS 和 $/TB 转向**延迟 QoS、带宽密度和每 token 吞吐的能效**。
- **采购量**随 GPU 部署规模增长，而非随数据增长。每个推理 Pod 无论应用数据量大小，都需要闪存层。
- **更新周期**可能短于传统 5 年企业级 SSD 生命周期，因为 KV cache 写入——即使经过过滤——仍会持续累积耐久性磨损。

### 6.2 每个 CMX 节点和 Pod 的预估 SSD 容量

基于已发布的参考架构和合作伙伴设计：

| 配置 | 每节点硬盘数 | 每节点容量 | 每 Pod 节点数 | 每 Pod 总闪存容量 |
|---------------|----------------|-------------------|---------------|-------------------|
| 大容量（QLC） | 5x 122 TB | ~614 TB | 4-8 | 2.5-5 PB |
| 均衡型（TLC） | 24x 25.6 TB | ~614 TB | 4-8 | 2.5-5 PB |
| 带宽优化（Gen6） | 20x 30.72 TB | ~614 TB | 4-8 | 2.5-5 PB |
| 分层（QLC + TLC） | 5x 122 TB + 8x 25.6 TB | ~815 TB | 4-8 | 3.3-6.5 PB |

即使以保守的采纳速度——到 2027 年底全球部署 10,000 个推理 Pod——CMX 将消耗 25-50 EB 的企业级 SSD 容量。作为参考，2025 年企业级 SSD 总出货量约为 120 EB([TrendForce Q3 2025 Enterprise SSD](https://www.trendforce.com/presscenter/news/20251205-12819.html))。仅 CMX 一项就可能占企业级 SSD 增量需求增长的 20-40%。

### 6.3 QLC TAM（总可寻址市场）扩展

CMX 从两个方面扩大了 QLC SSD 的 TAM：

1. **直接 CMX 部署。** 服务于温上下文层的 QLC 硬盘代表了 CMX 出现之前并不存在的全新增量需求。这些不是 HDD 替代（传统 QLC TAM 论述，详见[竞争格局 2.2 节](../../en/02-competitive-landscape.md)）——而是内存层级的扩展。

2. **级联置换效应。** 当 TLC SSD 被分配至 CMX 热上下文层后，这些 TLC 硬盘原来服务的存储工作负载（训练数据暂存、模型仓库、分析）转移到 QLC。这产生了独立于 CMX 本身的二次 QLC 需求。

综合效应对 QLC 供应商利好，尤其是 Solidigm（其战略完全聚焦 QLC）和 Micron（其 CuA 架构在 QLC 上具有结构性成本优势，详见[竞争格局 2.3 节](../../en/02-competitive-landscape.md)）。

### 6.4 采购模式变化

CMX 改变了 AI 基础设施中 SSD 的采购方式：

- **与 DPU/NIC 协议栈联合认证。** SSD 必须针对 BlueField-4 DPU 固件、Dynamo 软件和 Spectrum-X 网络进行验证。这在传统服务器级 SSD 认证之外增加了额外的认证周期。
- **功耗包络感知。** CMX 存储节点有严格的功耗预算。SSD 功耗（活跃态和空闲态）成为一级选型标准。Samsung PM1753 明确以低功耗为卖点；KIOXIA GP Series 部分出于能效考虑采用 XL-FLASH。
- **耐久性监控集成。** Dynamo 的磁盘卸载过滤会根据 SSD 健康状态动态调整写入行为。能通过标准化 NVMe 健康日志暴露详细 SMART 遥测数据（剩余耐久性、WAF（写放大因子）、温度）的 SSD 有助于 Dynamo 更好地优化。

---

## 7. 未来方向

### 7.1 CXL 挂载内存作为补充层

CXL Type 3 内存扩展器代表了主机 DRAM 和 CMX 闪存之间的一个潜在中间层。如[技术路线图第 4 节](../../en/05-technology-roadmap-2025-2030.md)所述，NAND 支撑的 CXL 设备呈现字节可寻址的内存接口，延迟在微秒级——大约比 DRAM 慢 10-50 倍，但对于小型随机访问可能比 NVMe 快 10-100 倍。

在配备 CMX 的 Pod 中，CXL 内存层可作为"G2.5"层：

| 层级 | 介质 | 延迟 | 每节点容量 | 角色 |
|------|--------|---------|-------------------|------|
| G1 | GPU HBM4 | ~1 us | 288 GB | 活跃 KV cache |
| G2 | 主机 DRAM | ~0.1-0.5 us | 1-2 TB | 热溢出 |
| G2.5 | CXL NAND 支撑 | 1-10 us | 2-8 TB | 温溢出（未来） |
| G3 | 本地 NVMe SSD | 10-80 us | 数百 TB | 温上下文 |
| G3.5 | CMX 闪存（Pod 级） | 20-100 us | PB 级 | 共享上下文 |

CXL 4.0 规范已于 2025 年 11 月发布，提供了该层所需的带宽（128 GT/s）和 Fabric 挂载内存语义。Samsung 的 CMM-D 2.0 CXL 内存模块（128-256 GB, CXL 2.0）已开始送样。然而，NAND 支撑的 CXL 在 CMX 环境中的实际部署是 2028 年之后的事，取决于 CPU 平台支持和软件栈的成熟度。

### 7.2 面向 CMX 的计算存储

计算存储设备（CSD）可在硬盘层面执行 KV cache 压缩或解压缩，从而减少 RDMA 带宽消耗并提高闪存有效容量，为 CMX 增加价值。如[技术路线图第 7 节](../../en/05-technology-roadmap-2025-2030.md)所述，ScaleFlux 的透明压缩方案已展示 2-4 倍的有效容量增益。应用于 QLC CMX 层，可将 614 TB 节点扩展至 1.2-2.4 PB 有效容量。

障碍在于集成复杂性：CMX 的软件栈（Dynamo、BlueField-4 DOCA）需要与 CSD 固件协调压缩/解压缩，引入延迟不确定性。这是 2028 年之后的机会。

### 7.3 FDP 和 ZNS 减少 CMX 写放大

FDP（Flexible Data Placement，灵活数据放置）和 ZNS（Zoned Namespaces，分区命名空间）与 CMX SSD 耐久性直接相关。如[技术路线图第 5 节](../../en/05-technology-roadmap-2025-2030.md)所述：

- **FDP** 允许 Dynamo 为 KV cache 写入标记放置提示（Reclaim Unit Handles），将短生命周期的上下文块与长生命周期的上下文块分离。这将写放大从 3-5 倍降低至 1.2-1.5 倍，有效将 QLC 耐久性提升两到三倍。
- **ZNS** 可实现更极致的 WAF 降低（接近 1.0 倍），但要求的顺序写入排序可能与 CMX 突发性、多流写入模式产生冲突。

FDP 是 CMX 更为务实的路径。如果 FDP 固件成为 CMX 认证 SSD 的标准配置——[技术路线图](../../en/05-technology-roadmap-2025-2030.md)预计在 2027-2028 年实现——QLC 与 TLC 的耐久性差距将显著缩小。一块启用 FDP 并实现 WAF 1.3 倍的 QLC 硬盘，其有效耐久性相当于未启用 FDP 时额定 DWPD 高 1.5-2 倍的 QLC 硬盘。

这对 QLC 在 CMX 中的角色具有变革性意义。借助 FDP，0.58 DWPD 的 QLC 硬盘可承受 0.75-0.85 DWPD 的有效写入量——覆盖除最极端 Agentic 工作负载外的所有场景。

---

## 8. 核心要点

1. **CMX 创造了真实的 SSD 新增需求。** 每个 Vera Rubin 推理 Pod 需要 PB 级 NVMe 闪存用于 KV cache 上下文存储。这不是对现有存储的替代，而是 2026 年之前不存在的全新存储层。

2. **CMX 的 SSD 需求特征独一无二。** 大块顺序写入、读取主导的检索、伴随持续换页的短暂数据、严格的延迟 QoS 要求。现有任何 SSD 工作负载类别都无法精确匹配。

3. **QLC 适用于 CMX，但非普遍适用。** QLC 的成本和容量优势对温上下文层（CMX 容量的 70-80%）极具吸引力。但热上下文层——受 Agentic 工作负载激进写入换页影响——需要 TLC 的耐久性。CMX 内部采用 QLC + TLC 分层策略是最务实的优化方案。

4. **Dynamo 的频率门控卸载对 QLC 至关重要。** 通过过滤写入、仅持久化复用频率 ≥ 2 的 KV 块，Dynamo 充当耐久性保护者。存储工程师应验证 Dynamo 的卸载策略是否针对其工作负载的会话特征进行了正确调优。

5. **FDP 将成为 CMX 中 QLC 的赋能技术。** 当 FDP 固件成为 CMX 认证 QLC SSD 的标配时（预计 2027-2028 年），写放大的降低将显著扩大 QLC 在 CMX 容量中的可寻址份额。

6. **PCIe Gen6 对 CMX 密度至关重要。** Micron 9650（Gen6, 28 GB/s）将每节点所需硬盘数量相比 Gen5 减半。随着 2027 年 Gen6 产品的普及，CMX 存储节点将变得更紧凑、更节能。

7. **供应商差异化是真实存在的。** KIOXIA CM9 和 GP Series、Micron 9650 和 6600 ION、Samsung PM1753、Solidigm D5-P5336/P5436 各自在 CMX 存储层级中占据不同定位。没有单一硬盘能覆盖全部需求。存储工程师必须根据其推理工作负载特征，在延迟-容量-耐久性的权衡面上进行评估。

8. **AI 推理的 SSD 采购已成为 GPU 基础设施决策。** CMX SSD 需与 BlueField-4、Dynamo 和 Spectrum-X 联合认证。存储工程团队必须与 GPU 基础设施团队紧密协作，使 SSD 选型与推理服务需求保持一致。

---

*报告信息截至 2026 年 3 月。基于公开可用的规格、NVIDIA GTC 2026 上的供应商发布信息以及本仓库中现有的 QLC SSD 研究分析。*
