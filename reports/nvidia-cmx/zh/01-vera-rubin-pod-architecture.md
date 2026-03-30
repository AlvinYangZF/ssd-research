# NVIDIA Vera Rubin Pod：面向存储工程师的架构深度解析

## 1. 引言

NVIDIA Vera Rubin 平台是 NVIDIA 有史以来最具雄心的 AI 基础设施架构。该平台于 GTC 2025 发布，并于 2026 年第一季度进入全面量产。Vera Rubin 不是单颗芯片或单台服务器——它是一个由 40 个机柜组成的 Pod 级 AI 超级计算机，将七种定制芯片设计与五种专用机柜级系统统一整合为一个协调运作的计算集群。

在核心指标上，Vera Rubin Pod 提供 60 EFLOPS 的 FP4 算力、10 PB/s 的总横向扩展带宽、1,152 颗 Rubin GPU、近 20,000 颗独立 NVIDIA 芯片以及 1.2 千万亿个晶体管。NVIDIA 宣称，与上一代 Blackwell 相比，推理性能最高提升 5 倍，每 token 成本降低 10 倍，每瓦性能提升 10 倍（[NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)）。

对于存储工程师而言，Vera Rubin Pod 的重要意义在于它正式在 AI 计算层次结构中引入了一个新的存储层级：CMX（Context Memory Extensions，上下文内存扩展）平台。该平台将 NVMe SSD 作为推理数据通路中的一等公民，专门用于 KV Cache（键值缓存）卸载。本报告逐一剖析七种芯片、五种机柜级系统以及连接它们的内存/带宽层次结构——并特别关注这些变化对存储基础设施规划的影响。

**时间线：** Vera Rubin NVL72 机柜于 CES 2026（2026 年 1 月）投入生产。云服务提供商和 OEM 合作伙伴的系统预计在 2026 年下半年出货。搭载 HBM4e 内存和全新 Kyber 机柜设计的 Rubin Ultra 计划于 2027 年推出（[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-launches-vera-rubin-nvl72-ai-supercomputer-at-ces-promises-up-to-5x-greater-inference-performance-and-10x-lower-cost-per-token-than-blackwell-coming-2h-2026)）。

---

## 2. 七种芯片

Vera Rubin 平台围绕七种不同的芯片设计构建，每种芯片都针对 Pod 内的特定功能进行了专门优化。这与此前 NVIDIA 主要提供 GPU 和 CPU 的模式截然不同；Vera Rubin 将网络、存储处理和交换视为同等重要的协同设计芯片。

### 2.1 Vera CPU

Vera CPU 是 NVIDIA 第二代数据中心 Arm 处理器，是 Grace 的继任者。关键规格如下：

| 规格 | 详情 |
|---|---|
| 核心数 | 88 个 Olympus 核心（NVIDIA 定制 Arm 设计，Armv9.2 ISA（指令集架构）） |
| 线程数 | 176 |
| IPC（每周期指令数）提升 | 相比 Grace（基于 Neoverse V2）提升 1.5 倍 |
| 内存 | 最高 1.5 TB LPDDR5X（通过 SOCAMM 模组） |
| 内存带宽 | 1.2 TB/s |
| I/O 接口 | PCIe Gen 6、CXL 3.1 |
| NVLink-C2C | 1.8 TB/s 裸片间互联（Grace 的 2 倍，即 900 GB/s 的 2 倍；PCIe 6.0 的 7 倍） |
| 精度支持 | 首款支持 FP8 的 CPU |

Olympus 核心并非标准 Arm Neoverse 设计——NVIDIA 在微架构层面进行了定制修改，针对 AI 相关工作负载进行了优化：强化学习编排、智能体推理、工具调用调度以及高吞吐数据预处理。NVIDIA 将 1.5 倍 IPC 提升描述为"相对于其他竞争架构的巨大代际跃迁"（[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-unveils-details-of-new-88-core-vera-cpus-positioned-to-compete-with-amd-and-intel-new-vera-cpu-rack-features-256-liquid-cooled-chips-that-deliver-up-to-a-6x-gain-in-cpu-throughput)）。

NVLink-C2C 互联以 1.8 TB/s 的速率运行，这一点尤为重要：它是"Vera Rubin 超级芯片"内部 Vera CPU 与配对 Rubin GPU 之间的连接。这条一致性裸片间链路意味着 CPU 和 GPU 共享统一内存空间，延迟远低于 PCIe 连接的配置方案。

对于存储工程师而言，PCIe Gen 6 和 CXL 3.1 的支持值得关注。CXL 3.1 实现了跨设备的内存池化和共享，这对未来 NVMe-oF（NVMe over Fabrics）和 CXL 直连存储层级如何集成到 CPU 地址空间具有重要意义。

### 2.2 Rubin GPU（R100/VR200）

Rubin GPU 是该平台的主计算引擎。关键规格如下：

| 规格 | 详情 |
|---|---|
| 制程 | TSMC 3nm |
| 晶体管数量 | 3,360 亿（Blackwell 2,080 亿的 1.6 倍） |
| 芯片架构 | 多芯片模组（MCM）：2 颗计算裸片 + 2 颗 I/O 裸片，基于 4 倍光罩 CoWoS-L 载板 |
| 流式多处理器 | 224 个 SM，搭载第六代 Tensor Core |
| 精度支持 | FP4、FP6、FP8、FP16、BF16、TF32、FP32、FP64 |
| HBM4（高带宽内存）容量 | 288 GB，分布在 8 个堆栈上 |
| HBM4 带宽 | 每 GPU 22 TB/s（Blackwell 8 TB/s 的 2.8 倍） |
| NVLink 6 带宽 | 每 GPU 3.6 TB/s（Blackwell 的 2 倍） |
| FP4 推理算力 | 每 GPU 50 PFLOPS（Blackwell 的 5 倍） |
| FP4 训练算力 | 每 GPU 35 PFLOPS（Blackwell 的 3.5 倍） |

每 GPU 22 TB/s 的 HBM4 带宽是对推理工作负载影响最大的单一指标。在此带宽下，单颗 R100 在长上下文推理时能够支撑更大的活跃 KV Cache，而不会造成计算流水线停滞。第三代 Transformer Engine 新增了硬件加速的自适应压缩功能，可使用针对 NVFP4 格式的两级微块缩放方案，在 Transformer 各层间动态调整精度（[NVIDIA Technical Blog](https://developer.nvidia.com/blog/inside-the-nvidia-rubin-platform-six-new-chips-one-ai-supercomputer/)）。

在 NVL72 机柜层面，72 颗 Rubin GPU 提供总计 20.7 TB 的 HBM4 容量和 260 TB/s 的 NVLink 6 横向扩展带宽。这构成了内存层次结构中的"热"层——即正在被计算处理的活跃数据。

### 2.3 Rubin Ultra GPU（2027 年）

虽然 Rubin Ultra 不属于 2026 年出货的当前 Vera Rubin Pod，但其路线图对容量规划具有参考价值：

| 规格 | 详情 |
|---|---|
| 芯片架构 | 4 颗计算 chiplet（基础 Rubin 的 2 倍） |
| HBM4e 内存 | 1 TB，分布在 16 个堆栈上（每堆栈 16 层 32Gb DRAM 裸片） |
| HBM4e 带宽 | 约 32 TB/s |
| FP4 推理算力 | 每 GPU 100 PFLOPS |
| TDP（热设计功耗） | 每 GPU 3.6 kW |
| 机柜设计 | Kyber 机柜，144 个 GPU 封装 |
| NVLink 版本 | NVLink 7（保持每端口 3.6 TB/s，增加 GPU 数量） |

从每 GPU 288 GB 跃升至 1 TB 的 HBM 容量将从根本上改变 KV Cache 存储分层的经济性。目前需要通过 CMX SSD 卸载的模型，可能在 Rubin Ultra 上完全存放在 HBM4e 中（[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-demonstrates-rubin-ultra-tray-worlds-1st-ai-gpu-with-1tb-of-hbm4e)）。

### 2.4 BlueField-4 DPU（数据处理器）

BlueField-4 是一个双裸片封装，作为 Pod 的基础设施处理器：

| 规格 | 详情 |
|---|---|
| CPU | 64 核 NVIDIA Grace（基于 Arm） |
| 网络 | 集成 ConnectX-9，800 Gb/s 吞吐量 |
| 算力提升 | 相比 BlueField-3 提升 6 倍 |
| 集群规模 | 支持比 BlueField-3 大 4 倍的 AI 工厂 |
| 核心功能 | 存储卸载、数据完整性、加密、NVMe 管理、网络虚拟化 |

在 CMX 存储机柜中，BlueField-4 是管理 NVMe SSD、运行存储服务以及处理 KV Cache 数据完整性和加密的核心引擎。每个 ICMSP（Inference Context Memory Storage Platform，推理上下文内存存储平台）存储机箱包含 4 颗 BlueField-4 DPU，每颗管理 150 TB 的 NVMe SSD 容量（[NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/)）。

对于存储工程师而言，BlueField-4 是关键组件：它是 NVMe SSD 与 GPU 网络之间的可编程数据通路。每颗 BlueField-4 内置的 64 核 Grace CPU 为内联数据服务提供了充足的算力——包括压缩、去重、加密以及 KV Cache 段的智能预取。

### 2.5 ConnectX-9 SuperNIC

| 规格 | 详情 |
|---|---|
| 吞吐量 | 每芯片 800 Gb/s |
| 横向扩展带宽 | 每 GPU 1.6 Tb/s（多颗 CX-9 芯片封装在一起） |
| 协议 | RoCEv2（RDMA over Converged Ethernet，融合以太网 RDMA） |
| 优化 | Spectrum-X 以太网网络加速 |

ConnectX-9 为跨机柜的 GPU 间东西向流量以及到存储的南北向流量提供横向扩展网络。每 GPU 1.6 Tb/s 的横向扩展带宽正是 CMX 存储机柜向远端 NVL72 机柜中的 GPU 传输 KV Cache 数据的管道（[NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)）。

### 2.6 NVLink 6 交换芯片

| 规格 | 详情 |
|---|---|
| 端口数 | 144 个 NVLink 端口 |
| 无阻塞交换容量 | 每交换芯片 14.4 TB/s |
| 每 GPU 带宽 | 3.6 TB/s 双向 |
| 机柜级带宽 | NVL72 整机柜 260 TB/s |
| SHARP 支持 | 通过 FP8 网络内归约（in-network reduction）实现 4 倍带宽效率 |

NVLink 6 交换芯片使 NVL72 机柜内的全部 72 颗 GPU 能够在全带宽全互联拓扑下通信。SHARP（Scalable Hierarchical Aggregation and Reduction Protocol，可扩展分层聚合归约协议）功能可直接在交换网络中执行 FP8 归约操作，将集合通信操作需要传输的数据量减少最高 4 倍（[NVIDIA NVLink](https://www.nvidia.com/en-us/data-center/nvlink/)）。

### 2.7 Spectrum-6 以太网交换芯片

| 规格 | 详情 |
|---|---|
| SN6810 交换容量 | 102.4 Tb/s，128 个 800G 端口，2U 外形尺寸 |
| SN6800 交换容量 | 409.6 Tb/s，512 个 800G 端口 |
| 架构 | 单芯片设计，集成 SerDes |
| 共封装光学 | 支持基于硅光子的互联 |

Spectrum-6 是连接 Pod 内五种机柜类型的横向扩展以太网网络。它为 CMX 存储访问、CPU 机柜通信和 Pod 间网络互联提供 RDMA 传输。Spectrum-X 软件栈通过自适应路由、拥塞控制和专为 AI 流量模式调优的遥测功能来优化 RoCE 性能（[NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/)）。

---

## 3. 五种机柜级系统

Vera Rubin Pod 并非同构集群。它由五种不同的机柜类型组成，每种都针对智能体 AI 推理和训练流水线中的特定角色进行了优化。五种机柜均基于第三代 NVIDIA MGX 模块化机柜架构，采用 100% 液冷和无线缆模块化托盘设计。

### 3.1 DGX NVL72——GPU 计算机柜

**配置：** 72 颗 Rubin GPU + 36 颗 Vera CPU（36 个 Vera Rubin 超级芯片），通过 NVLink 6 交换芯片互联。

**用途：** 预训练、后训练和推理的主计算引擎。每个 NVL72 机柜提供 3.6 EFLOPS 的 FP4 推理算力、20.7 TB 的 HBM4 内存以及 260 TB/s 的 NVLink 6 横向扩展带宽。

**设计：** 每个超级芯片（1 颗 Vera CPU + 2 颗 Rubin GPU）安装在 18 个计算托盘之一中。NVIDIA 声称无线缆模块化托盘设计使组装和维护速度比 Blackwell 快 18 倍。这是 NVIDIA 首个采用 100% 液冷的系统，完全取消了风扇。

**功耗：** 提供两种配置——Max Q 约 190 kW 每机柜（每 GPU 1.8 kW），Max P 约 230 kW 每机柜（每 GPU 2.3 kW）。

在完整的 40 机柜 Vera Rubin Pod 中，16 个 NVL72 机柜提供 1,152 颗 Rubin GPU，贡献了标称的 60 EFLOPS 算力（[NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)）。

### 3.2 Groq 3 LPX 机柜——解码加速

**配置：** 每机柜 256 颗 Groq 3 LPU（Language Processing Unit，语言处理单元），基于 MGX 基础设施构建。

**用途：** 面向万亿参数模型和百万 token 上下文的专用解码/推理加速。LPX 架构针对推理的自回归 token 生成阶段进行了优化，该阶段受内存带宽瓶颈制约，传统 GPU 计算利用率较低。

**意义：** 这是 NVIDIA 首次将非 GPU 加速器（Groq 的 LPU）集成到其平台中。LPX 机柜处理解码阶段，而 NVL72 机柜处理预填充和训练，形成异构推理流水线，在两个阶段均实现最优的每瓦吞吐量（[The Decoder](https://the-decoder.com/gtc-2026-with-groq-3-lpx-nvidia-adds-dedicated-inference-hardware-to-its-platform-for-the-first-time/)）。

### 3.3 Vera CPU 机柜——编排与强化学习

**配置：** 每机柜 256 颗 Vera CPU，液冷散热，基于 MGX 构建。

**用途：** 为强化学习（RL）训练循环、智能体推理编排、工具调用调度和通用预处理提供密集 CPU 算力。通过 Spectrum-X 以太网连接到 Pod。

**性能：** NVIDIA 宣称 CPU 吞吐量相比 Grace CPU 一代提升最高 6 倍。CPU 机柜为智能体 AI 工作流提供"大脑"——在这些工作流中，CPU 必须编排多步推理链、调用外部工具、管理状态，并向 GPU 和 LPU 机柜分发任务（[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-unveils-details-of-new-88-core-vera-cpus-positioned-to-compete-with-amd-and-intel-new-vera-cpu-rack-features-256-liquid-cooled-chips-that-deliver-up-to-a-6x-gain-in-cpu-throughput)）。

### 3.4 CMX / BlueField-4 STX 机柜——上下文内存存储

**配置：** BlueField-4 DPU 管理 NVMe SSD 阵列。每个 ICMSP 机箱包含 4 颗 BlueField-4 DPU，每颗 DPU 管理 150 TB 的 NVMe SSD 容量。

**用途：** 这是存储工程师最应关注的机柜类型。CMX 在 NVIDIA 的内存层次结构中建立了一个新的"G3.5"存储层级：

- **G3（本地）：** GPU 封装内的 HBM4——每 GPU 288 GB，22 TB/s
- **G3.5（CMX）：** 以太网连接的 NVMe 闪存，针对 KV Cache 优化——PB 级容量，通过 RDMA 访问
- **G4（共享）：** 传统共享企业存储（并行文件系统、对象存储）

CMX 充当 Pod 的"智能体长期记忆"。它存储因体积过大无法放入 HBM、但又需要以远低于传统存储的延迟进行访问的 KV Cache 数据。一个百万 token 上下文的多轮智能体对话的 KV Cache 很容易超过每会话 100 GB；乘以数千个并发会话，这远远超出了 HBM 的容量。

**性能指标：** CMX 与传统存储相比，可提供 5 倍的每秒 token 数（TPS）和 5 倍的能效提升。BlueField-4 DPU 运行内联存储服务——数据完整性校验、加密，以及在解码引擎需要之前将 KV Cache 段智能预载到 GPU HBM 中。

**网络：** Spectrum-X 以太网提供 RDMA 网络（RoCEv2），实现 GPU 机柜与 CMX 存储机柜之间的低延迟访问。从 SSD 到 GPU 的整个数据通路设计为无需计算侧 CPU 参与，通过 BlueField-4 的 DMA 引擎将数据直接传输（[NVIDIA CMX](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/)；[Blocks and Files](https://www.blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/4090541)）。

### 3.5 Spectrum-6 SPX 机柜——网络交换

**配置：** Spectrum-6 以太网交换机，采用 Spine/Leaf 拓扑。

**用途：** 将 Pod 内所有机柜类型连接为统一的超级计算机。提供横向扩展以太网网络，承载跨 NVL72 机柜的 GPU 间流量（补充机柜内的 NVLink）、CPU 机柜连接以及存储机柜访问。

**规格：** SPX 机柜使用 Spectrum-6 SN6810（102.4 Tb/s）和 SN6800（409.6 Tb/s）交换机，端口速率 800G。Spectrum-X 软件栈提供 AI 优化的自适应路由、拥塞控制和流调度（[NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/)）。

---

## 4. 内存与带宽层次结构

对于存储工程师而言，Vera Rubin Pod 最重要的方面是其多层级内存和带宽层次结构。每个层级具有不同的容量、带宽和延迟特征，决定了何种数据存放在何处。

### 4.1 层级概览

```
层级        介质            容量（每 GPU）        带宽（每 GPU）         延迟          范围
──────────  ──────────────  ──────────────────   ───────────────────   ────────────  ──────────────
HBM4        封装内          288 GB               22 TB/s               ~ns           GPU 本地
NVLink 6    铜缆/光纤      20.7 TB（机柜）      3.6 TB/s              ~us           机柜内
Spectrum-X  以太网 RDMA     Pod 范围             1.6 Tb/s (200 GB/s)  ~10s of us    跨机柜
CMX (SSD)   NVMe 闪存      每 BF4 150+ TB       受 SSD 限制           ~100s of us   Pod 级挂载
G4 存储     并行文件系统/对象存储  无上限          受网络限制            ~ms           数据中心级
```

### 4.2 HBM4：热层（每 GPU 22 TB/s）

HBM4 是唯一能够以全吞吐量馈送 Tensor Core 的内存层级。每 GPU 22 TB/s 的带宽使 Rubin GPU 拥有 Blackwell（8 TB/s）2.8 倍和 Hopper H100（3.35 TB/s）6.6 倍的内存带宽。每 GPU 288 GB 的容量（NVL72 机柜合计 20.7 TB）决定了无需卸载时推理的最大"工作集"。

对于长上下文大语言模型，KV Cache 随序列长度和批次大小线性增长。在大型 MoE（混合专家）模型上进行百万 token 上下文的单次推理会话，可消耗 50-100+ GB 的 KV Cache。在规模化运行时，KV Cache 主导了 HBM 的消耗，往往超过模型权重本身。

### 4.3 NVLink 6：纵向扩展网络（每机柜 260 TB/s）

NVLink 6 为每颗 GPU 提供 3.6 TB/s 的双向带宽——比 PCIe Gen 6 快 14 倍，比 NVLink 5（Blackwell）快 2 倍。在 NVL72 机柜内，全部 72 颗 GPU 通过 NVLink 6 交换网络以总计 260 TB/s 的聚合带宽通信。

这一带宽使 72 颗 GPU 能够作为具有 20.7 TB 统一 HBM4 内存的单个逻辑加速器运行。对于推理而言，这意味着 KV Cache 数据可以分布在全部 72 颗 GPU 上，并由任意 GPU 以 NVLink 速率访问，实际上在机柜内创建了一个 20.7 TB 的"共享"HBM 池。

NVLink 6 交换芯片的 SHARP 网络内归约能力对训练非常重要：梯度同步的 all-reduce 操作部分在交换网络内直接完成，减少了需要通过链路传输的数据量。

### 4.4 Spectrum-X / RDMA：横向扩展网络（每 GPU 1.6 Tb/s）

ConnectX-9 SuperNIC 通过 Spectrum-X 以太网网络为每颗 GPU 提供 1.6 Tb/s（200 GB/s）的横向扩展带宽。这是以下通信的传输层：

- **跨机柜 GPU 通信：** 当工作负载跨越多个 NVL72 机柜时，Spectrum-X 承载跨机柜流量。
- **CMX 存储访问：** GPU 机柜与 CMX 存储机柜之间的 KV Cache 读写通过该网络传输。
- **CPU 机柜通信：** Vera CPU 机柜与 GPU 机柜之间的智能体编排流量。

NVLink 6（3.6 TB/s）与 Spectrum-X（200 GB/s）之间的带宽差距约为 18 倍。这种不对称是有意为之的：NVLink 处理机柜内延迟敏感的高带宽计算通信，而 Spectrum-X 处理 Pod 范围内的批量数据传输和存储访问。

### 4.5 CMX：温存储层（NVMe 闪存）

CMX 是存储工程师最需要深入理解的新层级。它位于 HBM 与传统共享存储之间，专为一种工作负载而设计：智能体推理的 KV Cache。

**架构：** BlueField-4 DPU 管理 NVMe SSD 阵列。每颗 BlueField-4 管理 150 TB 的闪存。DPU 负责：

- RDMA 目标服务（将 SSD 以 RDMA 可访问内存的形式暴露给 GPU 网络）
- 数据完整性校验和校验和
- 静态和传输中加密
- 智能预取（在解码引擎需要之前将 KV Cache 段预载到 GPU HBM 中）
- 垃圾回收和磨损均衡协调

**访问模式：** KV Cache 卸载在预填充阶段（模型处理输入 token 并生成 KV 键值对时）是写密集型工作负载，在解码阶段（模型生成输出 token 并需要回读先前计算的 KV 键值对时）是读密集型工作负载。访问模式在单个会话内基本是顺序的，但跨会话则是随机的。

**为什么选择 SSD 而非 DRAM？** 经济性分析非常直观。大型模型单次百万 token 的 KV Cache 可达 50-100 GB。服务数千个并发智能体会话需要 PB 级的 KV Cache。以当前价格计算，使用 DRAM 或 HBM 在经济上不可行，但使用高性能 NVMe 闪存则完全可行。延迟代价（RDMA + SSD 的微秒级 vs. HBM 的纳秒级）对于 KV Cache 是可以接受的，因为解码阶段本质上是串行的——每个 token 依赖前一个 token，因此流水线天然存在停顿间隙，预取机制可以利用这些间隙来隐藏 SSD 延迟。

**NVIDIA 的定位：** NVIDIA 将 CMX 描述为建立了"一个新的 G3.5 层级——专门为 KV Cache 优化的以太网连接闪存层。该层级充当 AI 基础设施 Pod 的智能体长期记忆，容量足以同时容纳多个智能体的共享、持续演进的上下文，同时又足够'近'，可以频繁地将上下文预载回 GPU 和主机内存，而不会导致解码停滞"（[Blocks and Files](https://blocksandfiles.com/2026/01/06/nvidia-standardizes-gpu-cluster-kv-cache-offload-to-nvme-ssds/)）。

### 4.6 G4：传统共享存储

最外层级是传统数据中心存储——并行文件系统（Lustre、GPFS/Spectrum Scale、WEKA、VAST）、对象存储（兼容 S3）和网络附加存储。该层级负责：

- 训练数据集预载
- 检查点存储
- 模型权重分发
- 日志和遥测数据收集

到 G4 存储的带宽受数据中心网络限制，通常为每节点 100-400 Gb/s。该层级不在 Vera Rubin 的关键推理数据通路中；CMX 吸收了延迟敏感的 KV Cache 工作负载，否则这些工作负载将需要昂贵的高性能共享存储来承载。

---

## 5. 与前代产品的对比

### 5.1 Vera Rubin vs. Blackwell：关键指标

| 指标 | Blackwell（GB200 NVL72） | Vera Rubin（VR200 NVL72） | 提升幅度 |
|---|---|---|---|
| GPU 制程节点 | TSMC 4nm | TSMC 3nm | 1 代 |
| 每 GPU 晶体管数 | 2,080 亿 | 3,360 亿 | 1.6 倍 |
| HBM 类型 | HBM3e | HBM4 | 1 代 |
| 每 GPU HBM 容量 | 192 GB | 288 GB | 1.5 倍 |
| 每 GPU HBM 带宽 | 8 TB/s | 22 TB/s | 2.75 倍 |
| 每 GPU NVLink 带宽 | 1.8 TB/s | 3.6 TB/s | 2 倍 |
| NVLink 版本 | NVLink 5 | NVLink 6 | 1 代 |
| 机柜 NVLink 带宽 | 130 TB/s | 260 TB/s | 2 倍 |
| 每 GPU FP4 推理算力 | 约 10 PFLOPS | 50 PFLOPS | 5 倍 |
| 每 GPU FP4 训练算力 | 约 10 PFLOPS | 35 PFLOPS | 3.5 倍 |
| 机柜 HBM 容量 | 约 14 TB | 20.7 TB | 1.5 倍 |
| CPU | Grace（72 个 Neoverse V2 核心） | Vera（88 个 Olympus 核心） | +22% 核心数，1.5 倍 IPC |
| CPU 内存 | LPDDR5X | LPDDR5X | 同代 |
| CPU-GPU 互联 | NVLink-C2C 900 GB/s | NVLink-C2C 1.8 TB/s | 2 倍 |
| DPU | BlueField-3 | BlueField-4 | 6 倍算力 |
| NIC（网卡） | ConnectX-7（400G） | ConnectX-9（800G） | 2 倍 |
| 每 GPU 横向扩展带宽 | 400 Gb/s | 1.6 Tb/s | 4 倍 |
| 散热方式 | 风冷/液冷混合 | 100% 液冷 | 全面转型 |
| 每 token 推理成本 | 基准值 | 最高降低 10 倍 | 10 倍 |
| MoE 训练所需 GPU 数 | 基准值 | 减少 4 倍 | 4 倍 |
| 组装/维护时间 | 基准值 | 快 18 倍 | 18 倍 |

来源：[NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)；[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-launches-vera-rubin-nvl72-ai-supercomputer-at-ces-promises-up-to-5x-greater-inference-performance-and-10x-lower-cost-per-token-than-blackwell-coming-2h-2026)；[WCCFTech](https://wccftech.com/nvidia-rubin-most-advanced-ai-platform-50-pflops-vera-cpu-5x-uplift-vs-blackwell/)

### 5.2 架构差异

除原始数据之外，以下几个架构层面的变化使 Vera Rubin 与 Blackwell 形成了根本性区分：

**Pod 级设计取代服务器级设计。** Blackwell 主要被设计为可聚合成 SuperPOD 的机柜级系统（NVL72）。Vera Rubin 从一开始就被设计为包含五种异构机柜类型的 40 机柜 Pod，每种机柜针对特定功能优化。部署单元是 Pod，而非机柜。

**异构计算。** Groq 3 LPX 机柜（用于解码）和专用 Vera CPU 机柜（用于编排）的加入，意味着 Pod 拥有三种不同的计算引擎（GPU、LPU、CPU），每种引擎在其效率最高的推理流水线阶段处理任务。Blackwell 则所有任务都依赖 GPU。

**一等存储层级。** CMX/STX 机柜在 Blackwell 一代并不存在。KV Cache 向 SSD 卸载曾仅是软件层面的优化；而在 Vera Rubin 中，它成为硬件定义的层级，拥有专用芯片（BlueField-4）、专用机柜和专门构建的 RDMA 数据通路。

**100% 液冷。** Blackwell 支持风冷/液冷混合配置。Vera Rubin 全面采用液冷，实现了更高的每机柜功率密度，且 NVIDIA 宣称比传统蒸发冷却方案"耗水量大幅减少"（[CNBC](https://www.cnbc.com/2026/02/25/first-look-at-nvidias-ai-system-vera-rubin-and-how-it-beats-blackwell.html)）。

**无线缆模块化托盘设计。** 计算托盘无需线缆管理即可滑入滑出。NVIDIA 宣称组装和维护速度快 18 倍，这直接降低了数据中心运营成本和平均修复时间。

---

## 6. 存储工程师要点总结

### 6.1 KV Cache 现已成为存储问题

Vera Rubin Pod 架构最重要的一个结论是：NVIDIA 已正式宣告 KV Cache 管理是一个存储基础设施问题，而不仅仅是 GPU 内存管理问题。CMX 平台搭配 BlueField-4 管理的 NVMe SSD 创建了一个新的存储层级，存储团队必须对其进行规划、配置和运维。

具体影响如下：

- **SSD 采购：** CMX 机柜需要针对 KV Cache 卸载的混合读写模式优化的高耐久、高 IOPS NVMe SSD。标准的数据中心读密集型 SSD 可能不适用。NVIDIA 的合作伙伴生态系统（Solidigm、Samsung、SK hynix、Micron）将可能提供 CMX 认证硬盘。
- **容量规划：** KV Cache 大小随上下文长度、模型规模和并发会话数线性扩展。对于万亿级参数 MoE 模型的百万 token 上下文，每个活跃会话需规划 50-100 GB 的 KV Cache。一个服务 10,000 个并发会话的 Pod 需要 500 TB 到 1 PB 的 CMX 容量。
- **耐久性规划：** KV Cache 在预填充时写入，在解码时读取。在多轮智能体对话中，KV Cache 会被频繁更新。写放大和耐久性需求将高度依赖推理工作负载的组合。
- **网络规划：** CMX 访问运行在 Spectrum-X 以太网上，使用 RDMA。存储工程师必须规划以太网网络，确保 NVL72 GPU 机柜与 CMX 存储机柜之间有足够的带宽，并配置适当的 QoS（服务质量），以避免存储流量与跨机柜 GPU 流量之间的干扰。

### 6.2 G3.5 层级改变了存储层次结构

传统 AI 基础设施在推理方面有两个相关的存储层级：HBM（快速、容量小、昂贵）和共享存储（慢速、容量大、廉价）。CMX 引入了一个中间层级，改变了优化的经济性考量：

- 此前需要更多 GPU（以便将 KV Cache 放入 HBM）的模型，现在可以使用更少的 GPU 加上 CMX 存储，从而可能降低总体拥有成本（TCO）。
- NVIDIA 宣称的 MoE 训练所需 GPU 减少 4 倍，部分原因就是将中间状态卸载到 CMX。
- CMX 相比传统存储 5 倍的 TPS 提升表明，此前不具备 BlueField-4 加速和 RDMA 优化的基于 SSD 的 KV Cache 方案存在显著的性能差距。

### 6.3 BlueField-4 作为存储控制器

BlueField-4 不仅仅是一块网卡——它是一个配备 64 核 Grace CPU 的完整存储控制器。其算力超过许多独立存储阵列的配置。存储工程师应预期 BlueField-4 将运行复杂的存储服务：内联压缩、去重、加密、磨损均衡协调和预测性预取。BlueField-4 的可编程性意味着这些服务将在硬件生命周期内通过软件更新持续演进。

### 6.4 电力和冷却规划

一个完整的 40 机柜 Vera Rubin Pod 将消耗数兆瓦的电力。CMX 存储机柜也会增加电力预算。负责设施规划的存储工程师需要考虑：

- 所有机柜类型的 100% 液冷基础设施
- 计算机柜（每柜 190-230 kW）和存储机柜的配电
- 取消风冷简化了暖通空调，但需要可靠的液冷分配系统

### 6.5 Rubin Ultra 将改变层级边界

2027 年 Rubin Ultra 上市后，每 GPU 配备 1 TB HBM4e（每机柜 72 TB），许多当前溢出到 CMX 的工作负载将完全放入 HBM。然而，模型规模和上下文长度的增长速度快于 HBM 容量，因此即使有了 Rubin Ultra，CMX 仍将持续发挥作用。存储工程师应将 CMX 部署视为基础设施的持久组成部分，而非过渡方案。

### 6.6 Spectrum-X 以太网即存储网络

与 InfiniBand 时代的 HPC 存储不同，CMX 运行在以太网加 RDMA（RoCEv2）之上。这意味着存储工程师可以利用现有的以太网运维经验、工具链和监控体系。但性能要求极为严苛：800 Gb/s 链路、超低延迟以及无损以太网行为。Spectrum-X 软件栈提供了使之可行的 AI 专用优化（自适应路由、拥塞控制、流调度），但这要求使用 Spectrum-6 交换机——普通以太网交换机无法满足所需的性能要求。

---

## 参考来源

- [NVIDIA Vera Rubin Opens Agentic AI Frontier -- NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)
- [NVIDIA Vera Rubin POD: Seven Chips, Five Rack-Scale Systems, One AI Supercomputer -- NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)
- [Inside the NVIDIA Vera Rubin Platform: Six New Chips, One AI Supercomputer -- NVIDIA Technical Blog](https://developer.nvidia.com/blog/inside-the-nvidia-rubin-platform-six-new-chips-one-ai-supercomputer/)
- [Introducing NVIDIA BlueField-4-Powered Inference Context Memory Storage Platform -- NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/)
- [NVIDIA CMX Context Memory Storage Platform -- NVIDIA](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/)
- [NVIDIA Vera Rubin NVL72 -- NVIDIA](https://www.nvidia.com/en-us/data-center/vera-rubin-nvl72/)
- [NVIDIA Spectrum-X Ethernet Platform -- NVIDIA](https://www.nvidia.com/en-us/networking/spectrumx/)
- [NVIDIA NVLink -- NVIDIA](https://www.nvidia.com/en-us/data-center/nvlink/)
- [Examining Nvidia's 60 exaflop Vera Rubin POD -- Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidias-seven-chip-vera-rubin-platforms-turns-the-data-center-into-an-ai-factory)
- [Nvidia unveils details of new 88-core Vera CPUs -- Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-unveils-details-of-new-88-core-vera-cpus-positioned-to-compete-with-amd-and-intel-new-vera-cpu-rack-features-256-liquid-cooled-chips-that-deliver-up-to-a-6x-gain-in-cpu-throughput)
- [Nvidia launches Vera Rubin NVL72 AI supercomputer at CES -- Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-launches-vera-rubin-nvl72-ai-supercomputer-at-ces-promises-up-to-5x-greater-inference-performance-and-10x-lower-cost-per-token-than-blackwell-coming-2h-2026)
- [Nvidia demonstrates Rubin Ultra tray -- Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-demonstrates-rubin-ultra-tray-worlds-1st-ai-gpu-with-1tb-of-hbm4e)
- [NVIDIA Rubin Is The Most Advanced AI Platform -- WCCFTech](https://wccftech.com/nvidia-rubin-most-advanced-ai-platform-50-pflops-vera-cpu-5x-uplift-vs-blackwell/)
- [Nvidia pushes AI inference context out to NVMe SSDs -- Blocks and Files](https://blocksandfiles.com/2026/01/06/nvidia-standardizes-gpu-cluster-kv-cache-offload-to-nvme-ssds/)
- [NVIDIA STX and CMX: Infrastructure for Agentic AI Context Storage -- NAND Research](https://nand-research.com/nvidia-stx-cmx-infrastructure-for-agentic-ai-context-storage/)
- [GTC 2026: With Groq 3 LPX, Nvidia adds dedicated inference hardware -- The Decoder](https://the-decoder.com/gtc-2026-with-groq-3-lpx-nvidia-adds-dedicated-inference-hardware-to-its-platform-for-the-first-time/)
- [First look at Nvidia's AI system Vera Rubin -- CNBC](https://www.cnbc.com/2026/02/25/first-look-at-nvidias-ai-system-vera-rubin-and-how-it-beats-blackwell.html)
- [NVIDIA Vera Rubin NVL72 Detailed -- VideoCardz](https://videocardz.com/newz/nvidia-vera-rubin-nvl72-detailed-72-gpus-36-cpus-260-tb-s-scale-up-bandwidth)
