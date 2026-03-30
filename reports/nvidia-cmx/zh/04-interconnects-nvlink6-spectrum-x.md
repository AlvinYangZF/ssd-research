# 互连技术：NVLink 6 与 Spectrum-X

**面向存储工程师的 AI 推理基础设施技术架构分析**

## 摘要

NVIDIA Vera Rubin 平台引入了两个本质不同的互连层级——用于 GPU 间通信的 NVLink 6，以及用于机架间和 GPU 到存储互联的 Spectrum-X 以太网——二者共同定义了现代 AI 工厂的数据移动能力。对于存储工程师而言，理解这些互连技术并非可选项：它们决定了 CMX 存储架构中每一次 KV 缓存检索、每一次模型检查点加载、以及每一次上下文内存访问所能实现的吞吐量和延迟。本报告深入剖析两种互连技术，追踪从 GPU HBM 经由 NVLink、PCIe Gen6 和 Spectrum-X 以太网到 NVMe SSD 的完整数据路径，并识别存储系统设计必须考虑的带宽瓶颈。所有规格均反映截至 GTC 2026 公布的 Vera Rubin 平台信息。

---

## 1. 引言：为什么互连是解耦 AI 基础设施的核心

从单体 GPU 服务器向解耦式机架级 AI 系统的转变，已将互连从辅助组件转变为首要架构约束。在 Vera Rubin Pod——一个集成了 1,152 颗 Rubin GPU、Vera CPU、BlueField-4 DPU 和 NVMe 存储的 40 机架 AI 超级计算机中——数据必须穿越多个互联层级才能到达目的地。在代理式 AI 推理过程中，一次 KV 缓存查找可能始于 GPU 的执行流水线，经由 NVLink 域到达关联的 Vera CPU，通过 PCIe Gen6 到达 ConnectX-9 SuperNIC，穿越 Spectrum-X 以太网结构到达 BlueField-4 STX 存储节点，最终访问 NVMe 闪存。每一跳都会引入其固有的带宽上限和延迟开销。

对于存储工程师，系统设计中有两个核心问题：

1. **从 GPU 到 NVMe SSD 的端到端带宽是多少？** 答案决定了一个 CMX 存储机架在成为瓶颈之前能同时服务多少并发 KV 缓存检索请求。
2. **端到端延迟预算是多少？** 代理式 AI 推理对延迟高度敏感——每个 token 生成步骤都可能需要从闪存中检索上下文，而该检索必须在模型的 token 间时限内完成。

答案在于将 NVLink 6 和 Spectrum-X 不作为孤立技术来理解，而是作为层级化数据移动架构中的互补层级。

---

## 2. NVLink 6：GPU 间互联结构

### 2.1 规格与代际跃升

NVLink 6 随 Rubin GPU 架构一同推出，相比 Blackwell 上的 NVLink 5，每 GPU 互连带宽翻倍 ([NVIDIA NVLink](https://www.nvidia.com/en-us/data-center/nvlink/))：

| 参数 | NVLink 5 (Blackwell) | NVLink 6 (Rubin) |
|-----------|---------------------|-------------------|
| 每 GPU 带宽 | 1,800 GB/s | 3,600 GB/s |
| 每 GPU NVLink 端口数 | 18 | 36 |
| 每链路带宽 | 100 GB/s | ~100 GB/s（双向） |
| 机架级聚合带宽 (NVL72) | 130 TB/s | 260 TB/s |
| 相对 PCIe Gen6 | ~7x | ~14x |

每颗 Rubin R200 GPU 支持 36 条 NVLink 6 连接，提供 3.6 TB/s 的双向带宽——超过 PCIe Gen6 x16 链路带宽的 14 倍 ([NVIDIA Vera Rubin NVL72](https://www.nvidia.com/en-us/data-center/vera-rubin-nvl72/))。该带宽主要由混合专家模型（MoE）推理中的 all-to-all 集合操作和训练中的梯度同步所消耗。

### 2.2 NVLink 6 交换架构

NVLink 6 Switch 是 Vera Rubin 平台七款芯片之一。与在机架间运行的以太网或 InfiniBand 交换机不同，NVLink 交换机在单个机架内创建一个扁平化的、全对全的 GPU 通信域。

关键交换特性 ([Decoding Nvidia's Rubin Networking Math](https://www.wheelersnetwork.com/2025/11/decoding-nvidias-rubin-networking-math.html))：

- **交换托盘配置：** VR200 NVL72（也称为 NVL144，按单独 GPU 裸片计数）中的每个 NVLink 交换托盘包含多颗 NVLink 6 ASIC。相比 Blackwell 的 NVLink 5，Rubin 世代通过使用两倍数量的交换 ASIC 实现带宽翻倍。
- **无阻塞结构：** NVL72 机架内所有 72 颗 GPU 以全无阻塞、全对全拓扑连接。任何 GPU 都可以在不发生争用的情况下以全线速与任何其他 GPU 通信。
- **网内计算：** NVLink 交换机支持 NVIDIA SHARP（可扩展分层聚合和归约协议），精度为 FP8，使集合操作（allreduce、allgather）能够在交换结构内部分执行。每个交换托盘提供 14.4 TFLOPS 的 FP8 网内算力，减少了需要穿越整个互联结构的数据量。
- **聚合交换容量：** NVL72 的交换托盘在 72 GPU 域内共计提供 260 TB/s 的对分带宽。

### 2.3 多机架扩展：DGX SuperPOD

单个 NVL72 机架是基本的计算构建模块，但生产级 AI 工厂需要多机架规模。NVIDIA DGX SuperPOD with Vera Rubin NVL72 统一了八个 NVL72 系统——共 576 颗 Rubin GPU——提供高达 28.8 exaflops 的 FP4 性能和 600 TB 的聚合 HBM 容量 ([NVIDIA DGX SuperPOD](https://blogs.nvidia.com/blog/dgx-superpod-rubin/))。

NVL72 系统之间的机架间通信不使用 NVLink，而是依赖 Spectrum-X 以太网或 Quantum-X800 InfiniBand 作为横向扩展结构。这一架构边界——机架内使用 NVLink，机架间使用以太网/InfiniBand——对存储工程师至关重要：**存储流量始终经由以太网/InfiniBand 横向扩展结构传输，从不直接穿越 NVLink 域。**

### 2.4 与 NVLink 5 (Blackwell) 的对比

从 NVLink 5 到 NVLink 6 的代际提升主要体现在原始带宽而非架构重新设计 ([NVLink 6 Becomes the Backbone of Rubin Rack-Scale AI Architecture](https://convergedigest.com/nvlink-6-becomes-the-backbone-of-rubin-rack-scale-ai-architecture/))：

- **每 GPU 带宽翻倍：** 3,600 GB/s 对比 1,800 GB/s。通过将每 GPU 的 NVLink 端口数翻倍（36 对 18）和每托盘交换 ASIC 翻倍实现。
- **相同的逻辑拓扑：** NVLink 5 和 NVLink 6 均在 72 GPU NVL72 域内提供全对全连接。交换架构线性扩展。
- **提升的 MoE 吞吐量：** 对于 MoE 路由中关键的 all-to-all 操作，NVLink 6 相比 NVLink 5 提供高达 2 倍的吞吐量提升，直接惠及需要在专家子网络间路由 token 的推理工作负载。
- **相同的机架边界：** NVLink 5 和 NVLink 6 均不能原生扩展到单机架之外。多机架通信仍属以太网或 InfiniBand 的职责范围。

### 2.5 NVLink 在 CMX 数据路径中的角色

虽然 NVLink 本身不承载存储流量，但它在 CMX 上下文内存访问中发挥着间接但重要的作用。在 Vera Rubin 架构中，每颗 Rubin GPU 通过高带宽一致性互连（超级芯片封装内的 NVLink-C2C 芯片间链路，类似 Grace 架构）与一颗 Vera CPU 配对。当 GPU 需要访问存储在远程 CMX 存储节点上的 KV 缓存时：

1. GPU 发起内存访问请求，通过本地 NVLink-C2C 链路到达配对的 Vera CPU。
2. Vera CPU 的网络栈通过 ConnectX-9 SuperNIC 在 Spectrum-X 以太网上发起 RDMA 读取操作。
3. 返回数据路径反向执行上述流程。

计算托盘内 GPU 与 CPU 之间的 NVLink-C2C 链路以高带宽运行（Vera CPU 支持高达 1.2 TB/s 的内存带宽），但这一跳与上文讨论的 GPU 间 NVLink 6 互联结构是不同的。存储工程师不应将二者混淆。

---

## 3. Spectrum-X 以太网：横向扩展与存储互联结构

### 3.1 架构概述

NVIDIA Spectrum-X 是一个专为 AI 工作负载设计的以太网网络平台，由两个协同设计的组件组成 ([NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/))：

1. **Spectrum-6 以太网交换 ASIC** ——一颗 102.4 Tb/s 的单芯片交换机，搭载共封装光学（CPO）。
2. **ConnectX-9 SuperNIC** ——一款 800GbE 网络适配器，具备硬件 RDMA 卸载、PCIe Gen6 主机接口以及 AI 专用加速功能。

二者共同实现了 NVIDIA 所称的"以太网上的 InfiniBand 级性能"——具体来说，是基于融合以太网的高吞吐量 RDMA（RoCE，基于融合以太网的 RDMA），配合自适应路由和拥塞控制，接近传统 InfiniBand 的确定性行为。

### 3.2 Spectrum-6 交换 ASIC

Spectrum-6 是 NVIDIA 最新一代以太网交换芯片，与传统商用交换 ASIC 有着显著差异 ([Spectrum-6 Ethernet Switch Deep Dive](https://www.naddod.com/ai-insights/spectrum-6-ethernet-switch-deep-dive-sn6810-102-4t-and-sn6800-409-6t-switch))：

| 参数 | 规格 |
|-----------|--------------|
| 聚合交换容量 | 102.4 Tb/s |
| SerDes 速率 | 每通道 224G |
| 端口配置 | 512 x 200G 或 128 x 800G |
| 光学 | 共封装硅光子（32 x 1.6 Tb/s 光引擎） |
| 相比可插拔方案的功耗效率 | 光学功耗效率提升约 5 倍 |
| 相比可插拔方案的可靠性 | 光学可靠性提升约 10 倍 |

CPO（共封装光学）设计在架构层面意义重大。通过采用微环调制器技术将 32 个硅光子光引擎直接集成到交换机封装中，Spectrum-6 消除了传统设计中固有的信号完整性损耗——在传统方案中，224G SerDes 信号需要穿越长距离 PCB 走线到达前面板可插拔收发器。这不仅仅是效率的改进——在每通道 224G 速率下，交换 ASIC 与可插拔光模块之间的传统电气互连会引入足够大的信号衰减，以至于需要高功耗的中继器，或者不得不缩短可达链路距离。

Vera Rubin POD 中的 SPX（Spectrum Platform for X-scale）机架搭载这些 Spectrum-6 交换机，提供横向扩展以太网结构的脊干层，用于互连 NVL72 计算机架、STX 存储机架和 Vera CPU 机架 ([NVIDIA Vera Rubin POD](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/))。

### 3.3 ConnectX-9 SuperNIC

ConnectX-9 是 NVIDIA 最新的网络适配器，专为 AI 工厂部署而设计。它在计算节点（NVL72）和存储节点（STX）中均充当 Spectrum-X 以太网的端点 ([NVIDIA ConnectX-9 SuperNIC](https://docs.nvidia.com/networking/display/connectx9supernic))：

| 参数 | 规格 |
|-----------|--------------|
| 以太网速率 | 1 x 800GbE（单端口，无聚合） |
| 主机接口 | PCIe Gen6 x16（通过 Socket Direct 支持双路 x16） |
| RDMA | RoCEv2 硬件卸载 |
| 双路支持 | Socket Direct——将 32 通道 PCIe 总线分拆为两个 x16 接口 |
| 安全性 | 后量子密码学 (PQC)，CNSA 2.0 合规 |
| 外形尺寸 | OSFP |

800GbE 单端口能力值得关注：上一代 SuperNIC 需要多链路聚合才能达到相当的吞吐量。ConnectX-9 在单个光通道组上即可实现 800 Gb/s（100 GB/s），简化了布线并减少了交换机端口消耗。

PCIe Gen6 x16 主机接口提供约 128 GB/s 的双向带宽（Gen6 编码下每通道 64 GT/s x 16 通道 x 每次传输 2 字节）。这意味着 ConnectX-9 的 800GbE 线速（100 GB/s）完全在 PCIe Gen6 x16 的带宽范围之内，避免了早期世代中 NIC 线速可能超过 PCIe 带宽的主机侧瓶颈。

### 3.4 RoCE：基于融合以太网的 RDMA

Spectrum-X 相比普通以太网的性能优势源于其 RoCE 实现，远超基本的 RoCEv2 标准。该平台实现了三项关键增强 ([Turbocharging AI Workloads with Spectrum-X](https://developer.nvidia.com/blog/turbocharging-ai-workloads-with-nvidia-spectrum-x-networking-platform/))：

**RoCE 自适应路由：** 传统以太网使用静态 ECMP（等价多路径）哈希将流量分散到可用路径上。当哈希碰撞将多条大流集中在同一链路时，会产生持续的热点。Spectrum-X 在交换机层面实现了细粒度的逐包自适应路由。Spectrum-6 交换机为每个数据包动态选择拥塞最少的路径，在所有可用互联路径上实现接近最优的负载均衡。这在功能上等同于 InfiniBand 的自适应路由能力——标准以太网交换机不具备此功能。

**RoCE 拥塞控制：** Spectrum-X 采用基于遥测的拥塞控制机制，交换机硬件实时监控队列深度和链路利用率，然后通知 SuperNIC 调整发送端注入速率。这种硬件到硬件的反馈环路以微秒级粒度运行，远快于依赖 ECN 标记和端到端反馈的 DCQCN 等软件拥塞控制方案。

**RoCE 性能隔离：** 在多租户或混合工作负载环境中，Spectrum-X 提供流量隔离，防止一条流影响另一条流的性能。这在 AI 工厂中至关重要，因为存储流量（CMX KV 缓存检索）与机架间 GPU 通信流量共享以太网结构。

这些增强的综合效果十分显著：NVIDIA 报告在大规模、有负载的条件下可实现高达 95% 的有效带宽利用率，而普通 RoCE 结构通常仅为 50-70%。在使用 Israel-1 超级计算机进行的存储专项测试中，Spectrum-X 相比标准以太网将读取带宽提升了最高 48%，写入带宽提升了最高 41% ([Accelerating AI Storage with Spectrum-X](https://developer.nvidia.com/blog/accelerating-ai-storage-by-up-to-48-with-nvidia-spectrum-x-networking-platform-and-partners/))。

### 3.5 Spectrum-XGS：千兆级扩展

对于跨越多个数据中心建筑或园区的部署，NVIDIA 推出了 Spectrum-XGS——一种跨域扩展技术，将分布式数据中心统一为单一逻辑 AI 互联结构 ([NVIDIA Spectrum-XGS](https://nvidianews.nvidia.com/news/nvidia-introduces-spectrum-xgs-ethernet-to-connect-distributed-data-centers-into-giga-scale-ai-super-factories))。虽然这对单 Pod 存储架构的直接相关性较低，但对于设计地理分布式上下文内存池（KV 缓存数据可能需要跨物理站点访问）的存储工程师而言，Spectrum-XGS 具有重要意义。

---

## 4. CMX 数据路径分析：GPU 到 NVMe SSD

本节追踪从 Rubin GPU 到 CMX 存储节点中 NVMe SSD 的 KV 缓存检索完整数据路径，并识别每一跳的带宽上限和近似延迟贡献。这是与存储工程师最直接相关的分析。

### 4.1 完整路径

```
Rubin GPU (HBM4)
  │
  │  NVLink-C2C（芯片间一致性链路）
  │  每方向 ~900 GB/s
  ▼
Vera CPU
  │
  │  PCIe Gen6 x16
  │  ~128 GB/s 双向（每方向 ~64 GB/s）
  ▼
ConnectX-9 SuperNIC
  │
  │  Spectrum-X 以太网 (800GbE RoCE)
  │  每端口 100 GB/s 线速
  ▼
Spectrum-6 Switch (SPX 机架)
  │
  │  Spectrum-X 以太网 (800GbE RoCE)
  │  每端口 100 GB/s 线速
  ▼
BlueField-4 DPU (STX 存储节点)
  │
  │  PCIe Gen6 至 NVMe 控制器
  │  每个 PCIe Gen6 x16 根复合体 ~128 GB/s
  ▼
NVMe SSD（例如 Micron 9650）
  每驱动器 ~28 GB/s 顺序读取
```

### 4.2 每一跳的带宽分析

**第 1 跳：GPU 到 Vera CPU (NVLink-C2C)**
Rubin GPU 和 Vera CPU 通过计算托盘内的芯片间一致性链路连接。在满配 SOCAMM2 配置下，Vera CPU 支持高达 1.2 TB/s 的内存带宽。GPU 与 CPU 之间的 C2C 链路带宽不是存储访问的瓶颈——其设计容量远高于存储需求，主要满足 GPU 内存访问模式的更高带宽要求。

**瓶颈评估：** 不构成存储流量的约束。C2C 链路有充足的余量。

**第 2 跳：Vera CPU 到 ConnectX-9 SuperNIC (PCIe Gen6 x16)**
PCIe Gen6 x16 宽度提供约 128 GB/s 双向带宽（每方向 64 GB/s）。ConnectX-9 的 800GbE 端口以 100 GB/s 线速运行。由于 100 GB/s > 64 GB/s（单方向），单个 PCIe Gen6 x16 通道无法完全填满单方向的 NIC 带宽。但是，ConnectX-9 的 Socket Direct 特性可将 32 通道 PCIe 总线分拆为两个 x16 接口，在双路配置下可能提供每方向 128 GB/s 的带宽。

**瓶颈评估：** 在单路配置中，PCIe Gen6 x16（每方向 64 GB/s）比 NIC 的 100 GB/s 线速更为紧张。实际使用中，RDMA 操作是双向的（命令发出，数据返回），因此 128 GB/s 的聚合带宽对大多数访问模式而言是充足的。此跳构成中等程度的约束。

**第 3 跳：ConnectX-9 到 Spectrum-6 Switch (800GbE 以太网)**
800GbE 链路提供 100 GB/s 的原始带宽。在 RoCE 协议开销（包头、CRC、包间间隔）影响下，大块 RDMA 传输的有效吞吐量约为 95-97 GB/s。对于小型随机 KV 缓存读取（4KB-64KB），每消息的开销显著降低了有效吞吐量——NIC 的 IOPS 上限成为限制因素，而非原始带宽。

**瓶颈评估：** 对于大型顺序 KV 缓存传输（例如长上下文预填充），800GbE 链路是充足的。对于小型随机访问，RDMA 消息速率和 NIC IOPS 成为约束。

**第 4 跳：Spectrum-6 Switch 转发**
Spectrum-6 交换机以 102.4 Tb/s 的聚合容量运行，采用直通转发。每端口穿越交换机的延迟在低百纳秒级别。对于任何单条流而言，交换机本身不构成带宽瓶颈——其容量设计可同时处理数百条 800GbE 流。

**瓶颈评估：** 不构成约束。交换级拥塞由自适应路由和拥塞控制管理。

**第 5 跳：Spectrum-6 Switch 到 BlueField-4 DPU (800GbE 以太网)**
与第 3 跳对称。STX 存储节点中的 BlueField-4 DPU 终结 RoCE 连接并将其转换为本地 NVMe 操作。

**瓶颈评估：** 与第 3 跳相同。

**第 6 跳：BlueField-4 到 NVMe SSD (PCIe Gen6)**
BlueField-4 直接管理 NVMe SSD，以线速处理 NVMe-oF 终结、数据完整性（CRC）和加密。Micron 9650 等 PCIe Gen6 SSD 每盘提供高达 28 GB/s 的顺序读取吞吐量。单个 STX 存储节点在 BlueField-4 的 PCIe 根复合体后聚合多块 SSD。

**瓶颈评估：** 单块 SSD 带宽（28 GB/s）是整条路径中最窄的节点。CMX 存储节点必须在多块 SSD 上并行条带化读取，才能填满上游 800GbE 链路。在 100 GB/s 有效网络带宽下，大约需要 4 块 SSD 以峰值顺序读取速率并行读取才能填满管道。在实际使用中，由于 KV 缓存访问的典型随机读取模式，需要更多 SSD 来补偿高队列深度下每盘吞吐量的下降。

### 4.3 KV 缓存检索的延迟预算

估算从 GPU 到 NVMe 再返回的单次 KV 缓存读取端到端延迟：

| 跳 | 组件 | 估算延迟 |
|-----|-----------|-------------------|
| 1 | GPU 到 Vera CPU (C2C) | < 100 ns |
| 2 | CPU 到 ConnectX-9 (PCIe + 驱动) | 500 ns - 1 us |
| 3 | ConnectX-9 RDMA 处理 | 1 - 2 us |
| 4 | 网络传输（交换机 + 线缆） | 1 - 3 us |
| 5 | BlueField-4 RDMA 终结 | 1 - 2 us |
| 6 | NVMe SSD 访问（4KB 随机读取） | 50 - 100 us |
| 7 | 返回路径（第 5-1 跳反向） | 3 - 6 us |
| **合计** | **端到端** | **~60 - 120 us** |

NVMe SSD 的访问延迟主导了整个预算，当代 QLC 或 TLC 闪存上 4KB 随机读取的延迟为 50-100 微秒。所有网络和互连跳加在一起贡献约 10-15 微秒——不到总延迟的 20%。这对存储工程师有一个关键启示：**超过一定程度后继续优化网络结构的收益递减，SSD 介质延迟才是主导因素。** 在 SSD 层面的投入（更高队列深度、智能预取、针对热 KV 条目的 SLC 缓存、用于解压缩的计算存储）比进一步的网络调优能带来更多的延迟降低。

### 4.4 RDMA 零拷贝访问

CMX 数据路径的一个关键性能特性是 RDMA 零拷贝操作。在传统存储访问中：

1. 数据从 SSD 读入 DPU 内存。
2. 数据被复制到网络缓冲区。
3. 数据通过网络传输。
4. 数据到达主机 NIC 缓冲区。
5. 数据被复制到应用程序内存。

使用 Spectrum-X 上的 RDMA 零拷贝：

1. 数据从 SSD 读入 BlueField-4 的已注册内存区域。
2. 请求端的 ConnectX-9 发起 RDMA READ 操作，将数据直接从 BlueField-4 的内存传输到 GPU 可访问的内存区域——完全绕过 Vera CPU 在数据路径中的参与。
3. GPU 从已注册内存区域访问数据。

这消除了多次内存拷贝和 CPU 中断，降低了延迟和 CPU 开销。BlueField-4 的 KV I/O 平面软件专门设计为将热 KV 缓存条目保持在 DRAM 中进行 RDMA 服务，仅在缓存未命中时才回退到 NVMe 闪存。这种分层访问模式意味着频繁访问的上下文内存可以以 DRAM 速度（约 1-5 us 往返）而非闪存速度（约 60-120 us）提供服务。

---

## 5. NVLink 与 Spectrum-X：互补角色

一个常见的误解是 NVLink 和 Spectrum-X 是相互竞争的互连技术。它们在 Vera Rubin 架构中服务于根本不同的目的：

| 维度 | NVLink 6 | Spectrum-X 以太网 |
|-----------|----------|-------------------|
| **主要功能** | 机架内 GPU 间通信 | 机架间、GPU 到存储、GPU 到 CPU 机架 |
| **拓扑** | NVL72 72 GPU 域内全对全 | 跨机架的胖树或 Clos 拓扑 |
| **每 GPU 带宽** | 3,600 GB/s | 100-800 GB/s（取决于每节点 NIC 数量） |
| **协议** | 专有 NVLink | 标准以太网 (RoCEv2) |
| **延迟** | GPU 间亚微秒 | 低个位数微秒 |
| **可扩展性** | 72 GPU（单机架） | 数千端点（多机架） |
| **存储访问** | 否（不连接存储） | 是（主要存储访问结构） |
| **标准合规性** | NVIDIA 专有 | IEEE 802.3 以太网 |

### 5.1 职责分工

在 Vera Rubin POD 中，工作负载流量自然分布在这两层互联结构上：

**NVLink 6 承载：**
- MoE 模型中的全对全专家路由
- 训练期间的梯度同步 (allreduce)
- 节点内 GPU 间的张量并行通信
- 机架内各阶段间的流水线并行激活值传输

**Spectrum-X 承载：**
- 从 CMX 存储节点（STX 机架）检索 KV 缓存
- 从持久存储加载模型权重
- 数据并行的机架间 GPU 通信
- 来自 Vera CPU 机架的 RL（强化学习）编排流量
- 管理、遥测和控制面流量

### 5.2 对存储工程师的启示

核心洞察是：**所有存储流量都仅通过 Spectrum-X 以太网传输**。尽管 NVLink 拥有远高得多的带宽，但它对存储子系统而言是不可见的。这意味着：

1. **存储带宽规划应使用 Spectrum-X 的数据，而非 NVLink 的数据。** NVL72 机架的聚合存储带宽取决于 ConnectX-9 SuperNIC 的数量及其到 SPX 结构的上行链路，而非 260 TB/s 的 NVLink 域。

2. **CMX 存储节点的规模应匹配 Spectrum-X 结构带宽。** 每个 STX 存储机架提供的 NVMe SSD 容量和吞吐量与其连接到 SPX 脊干层的 800GbE 链路相匹配。超过网络交付能力地过度配置 SSD 吞吐量是浪费。

3. **互联结构拥塞对存储的影响大于对计算的影响。** 由于 NVLink 处理了机架内最高带宽的计算流量，Spectrum-X 结构承受的负载相对较低。然而，在上下文检索密集的推理服务期间，存储流量可能主导以太网结构。Spectrum-X 的自适应路由和拥塞控制对于在负载下维持可预测的存储访问延迟至关重要。

---

## 6. NetX 机架级网络：连接整个 Pod

### 6.1 五种机架类型

Vera Rubin POD 定义了五种不同的机架级系统，均基于第三代 NVIDIA MGX 模块化架构构建 ([NVIDIA Vera Rubin POD](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/))：

| 机架类型 | 功能 | 关键芯片 | POD 内数量 |
|-----------|----------|-------------|-----------------|
| **DGX Vera Rubin NVL72** | GPU 计算（训练 + 推理） | 72 颗 Rubin GPU、36 颗 Vera CPU、NVLink 6 Switch | 16 个机架（8 个 SuperPOD） |
| **LPX** | 解码加速 | 256 颗 Groq 3 LPU | 可变 |
| **Vera CPU Rack** | RL 编排、数据处理 | 256 颗 Vera CPU | 可变 |
| **STX** | CMX 上下文内存存储 | BlueField-4 DPU、NVMe SSD | 可变 |
| **SPX** | 以太网网络脊干层 | 搭载 CPO 的 Spectrum-6 交换机 | 可变 |

完整的 Vera Rubin POD 包含约 40 个机架，总计 1,152 颗 Rubin GPU、近 20,000 颗 NVIDIA 裸片、1.2 千万亿个晶体管、60 exaflops 的 FP4 算力和 10 PB/s 的聚合带宽。

### 6.2 SPX 作为互联结构脊干层

SPX 网络机架作为整个 Pod 的核心互连枢纽。它搭载具有 102.4 Tb/s 容量和共封装光学的 Spectrum-6 交换机，连接：

- **NVL72 <-> NVL72：** 用于数据并行、跨机架流水线并行和多机架集合操作的机架间 GPU 通信。
- **NVL72 <-> STX：** GPU 到 CMX 存储的流量，用于 KV 缓存检索、上下文内存访问和检查点 I/O。
- **NVL72 <-> Vera CPU Rack：** GPU 计算节点与基于 CPU 的 RL 训练、数据预处理和编排之间的通信。
- **NVL72 <-> LPX：** 在预填充（GPU）和解码（LPU）阶段之间路由推理请求。
- **STX <-> 外部存储：** 访问 Pod 之外的持久存储（模型仓库、训练数据集）。

### 6.3 网络带宽预算

对存储工程师而言，一个有用的练习是估算每个 NVL72 机架的可用存储带宽：

- 每个 NVL72 机架包含多个用于 Spectrum-X 上行链路的 ConnectX-9 SuperNIC（确切数量取决于配置，但 NVIDIA 参考设计为每个计算托盘配置了多条 800GbE 上行链路）。
- 每个 NVL72 包含 36 个计算托盘（每个包含 2 颗 GPU 和 1 颗 CPU），假设每个托盘至少有一条 800GbE 上行链路在横向扩展计算流量和存储流量之间共享，则每 GPU 的以太网带宽约为 50-100 GB/s。
- 但是，该带宽在机架间计算流量和存储访问之间共享。在推理服务期间，由于机架间计算流量低于训练时的水平，更多以太网带宽可用于 CMX 存储访问。

实际意义是：运行推理工作负载的 NVL72 机架能够向 CMX 层维持数百 GB/s 范围的聚合存储读取吞吐量，这必须由 SPX 结构后面充足的 STX 存储节点配置来匹配。

### 6.4 七款芯片，一个统一结构

Vera Rubin 平台中的七款芯片——Rubin GPU、Vera CPU、NVLink 6 Switch、ConnectX-9 SuperNIC、BlueField-4 DPU、Spectrum-6 Switch 和 Groq 3 LPU——经协同设计以作为一个统一系统运作 ([NVIDIA Vera Rubin Platform](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform))。网络栈是统一的：

- **NVLink 6** 提供机架内 GPU 互联结构。
- **ConnectX-9** 在计算和 CPU 节点上提供主机到以太网的桥接。
- **Spectrum-6** 提供以太网交换结构。
- **BlueField-4** 提供存储侧以太网终结和 NVMe 管理。

这种垂直整合意味着 NVIDIA 控制着从 GPU 到 SSD 的完整数据路径，实现了商用组件无法达到的端到端优化。RDMA 语义在整条路径上得以保持。遥测数据从每个组件流回集中式 AI 驱动的网络管理系统。拥塞信号在硬件层面从交换机传播到 NIC，无需软件参与。

---

## 7. 关键要点

### 面向存储系统架构师

1. **NVMe SSD 是延迟瓶颈，而非网络。** 在 CMX 数据路径中，NVMe 闪存访问（随机读取 50-100 us）占端到端延迟的 80-90%。网络各跳合计仅贡献 10-15 微秒。应投资于 SSD 层优化：更高队列深度、智能预取、针对热条目的 SLC 缓存，以及 Micron 9650 等 PCIe Gen6 驱动器。

2. **存储带宽受限于 Spectrum-X，而非 NVLink。** 260 TB/s 的 NVLink 域与存储规划无关。应按照所服务计算机架的 800GbE Spectrum-X 上行链路容量来规划 CMX 存储机架大小。大约 4 块 PCIe Gen6 NVMe SSD 以峰值顺序读取速率即可填满一条 800GbE 链路。

3. **RDMA 零拷贝不是可选项——它是设计前提。** CMX 的性能目标假定 RDMA 零拷贝数据移动。引入额外内存拷贝的存储方案（例如用户态缓冲、内核态 NVMe-oF 目标实现）将无法满足延迟目标。BlueField-4 的硬件 KV I/O 平面正是为此而设计。

4. **Spectrum-X 的自适应路由和拥塞控制对存储 QoS 至关重要。** 与专用存储网络不同，Spectrum-X 结构承载混合流量（计算 + 存储）。如果没有自适应路由和硬件拥塞控制，存储读取的尾延迟将不可预测。Spectrum-X 的逐包负载均衡和基于遥测的速率控制使共享结构上的存储访问成为可能。

### 面向容量规划

5. **STX 机架数量随推理上下文需求扩展。** 每个 STX 机架提供有限的 NVMe 容量和吞吐量。随着代理式 AI 工作负载增长上下文窗口（每会话数百万 token），STX 存储机架与 NVL72 计算机架的比例将增加。NVIDIA 声称 CMX 相比传统存储可实现 5 倍 tokens-per-second 提升，前提是配置了充足的 STX 容量。

6. **PCIe Gen6 是一个过渡性瓶颈。** 对于 x16 链路每方向 64 GB/s 的带宽，PCIe Gen6 在单方向上比 100 GB/s 的 800GbE 线速更窄。未来世代（PCIe Gen7、CXL 3.0）将拓宽这一限制，但对于 2026-2027 年的部署，PCIe Gen6 是系统设计师必须仔细平衡 NIC 端口数、SSD 数量和 PCIe 通道分配的关键点。

7. **Spectrum-6 的共封装光学改变了部署经济性。** CPO 相比可插拔光学方案在功耗效率上提升 5 倍、可靠性提升 10 倍，降低了大规模以太网结构的运营成本。对于需要将数十个 STX 机架连接到数百个计算节点的存储网络，CPO 使密集、高基数交换拓扑在经济上变得可行。

### 全局视角

Vera Rubin 平台的互连架构围绕清晰的层级设计：NVLink 6 在 72 GPU 计算域内提供海量带宽（每 GPU 3.6 TB/s），而 Spectrum-X 以太网在包括存储在内的更广泛 Pod 范围内提供灵活的、基于标准的连接（每端口 800GbE）。对于存储工程师，Spectrum-X 是决定 CMX 性能的技术——其 RoCE 实现、自适应路由和 BlueField-4 集成定义了每次 KV 缓存访问所能实现的吞吐量和延迟。NVLink 域是计算层面的关注点；以太网域才是存储所在之处。

---

*本报告引用的来源：*
- [NVIDIA NVLink & NVSwitch Platform](https://www.nvidia.com/en-us/data-center/nvlink/)
- [NVIDIA Vera Rubin NVL72](https://www.nvidia.com/en-us/data-center/vera-rubin-nvl72/)
- [NVIDIA Spectrum-X Platform](https://www.nvidia.com/en-us/networking/spectrumx/)
- [ConnectX-9 SuperNIC Documentation](https://docs.nvidia.com/networking/display/connectx9supernic)
- [Spectrum-6 Switch Deep Dive — NADDOD](https://www.naddod.com/ai-insights/spectrum-6-ethernet-switch-deep-dive-sn6810-102-4t-and-sn6800-409-6t-switch)
- [Turbocharging AI Workloads with Spectrum-X — NVIDIA Technical Blog](https://developer.nvidia.com/blog/turbocharging-ai-workloads-with-nvidia-spectrum-x-networking-platform/)
- [Accelerating AI Storage with Spectrum-X — NVIDIA Technical Blog](https://developer.nvidia.com/blog/accelerating-ai-storage-by-up-to-48-with-nvidia-spectrum-x-networking-platform-and-partners/)
- [Decoding Nvidia's Rubin Networking Math — Wheeler's Network](https://www.wheelersnetwork.com/2025/11/decoding-nvidias-rubin-networking-math.html)
- [NVIDIA Vera Rubin POD — NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)
- [NVIDIA Vera Rubin Platform Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)
- [NVLink 6 Backbone of Rubin Architecture — Converge Digest](https://convergedigest.com/nvlink-6-becomes-the-backbone-of-rubin-rack-scale-ai-architecture/)
- [NVIDIA DGX SuperPOD for Rubin](https://blogs.nvidia.com/blog/dgx-superpod-rubin/)
- [NVIDIA Spectrum-XGS Ethernet](https://nvidianews.nvidia.com/news/nvidia-introduces-spectrum-xgs-ethernet-to-connect-distributed-data-centers-into-giga-scale-ai-super-factories)
- [NVIDIA BlueField-4 CMX Storage Platform — NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/)
- [Micron 9650 PCIe Gen6 SSD — StorageReview](https://www.storagereview.com/news/micron-begins-volume-production-of-hbm4-pcie-gen6-ssd-and-192gb-socamm2-for-nvidia-vera-rubin)
