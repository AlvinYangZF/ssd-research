# NVIDIA Vera Rubin Pod: Architecture Deep Dive for Storage Engineers

## 1. Introduction

The NVIDIA Vera Rubin platform represents the most ambitious AI infrastructure architecture NVIDIA has ever assembled. Announced at GTC 2025 and entering full production in Q1 2026, Vera Rubin is not a single chip or a single server -- it is a 40-rack, pod-scale AI supercomputer that unifies seven custom silicon designs across five purpose-built rack-scale systems into a single coherent computing fabric.

At its headline numbers, the Vera Rubin Pod delivers 60 exaflops of FP4 compute, 10 PB/s of total scale-up bandwidth, 1,152 Rubin GPUs, nearly 20,000 individual NVIDIA dies, and 1.2 quadrillion transistors. NVIDIA claims up to 5x inference performance and 10x lower cost per token compared to the preceding Blackwell generation, with 10x more performance per watt ([NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)).

For storage engineers, the Vera Rubin Pod is significant because it formally introduces a new storage tier into the AI compute hierarchy: the CMX (Context Memory Extensions) platform, which places NVMe SSDs as a first-class citizen in the inference data path for KV cache offload. This report examines each of the seven chips, the five rack-scale systems, and the memory/bandwidth hierarchy that connects them -- with particular attention to what this means for storage infrastructure planning.

**Timeline:** The Vera Rubin NVL72 rack entered production at CES 2026 (January 2026). Partner systems from cloud providers and OEMs are expected to ship in H2 2026. Rubin Ultra, with HBM4e memory and the new Kyber rack design, is planned for 2027 ([Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-launches-vera-rubin-nvl72-ai-supercomputer-at-ces-promises-up-to-5x-greater-inference-performance-and-10x-lower-cost-per-token-than-blackwell-coming-2h-2026)).

---

## 2. The Seven Chips

The Vera Rubin platform is built around seven distinct silicon designs, each purpose-built for a specific function within the pod. This is a departure from earlier generations where NVIDIA shipped primarily a GPU and a CPU; Vera Rubin treats networking, storage processing, and switching as equally critical co-designed silicon.

### 2.1 Vera CPU

The Vera CPU is NVIDIA's second-generation data center Arm processor, succeeding Grace. Key specifications:

| Specification | Detail |
|---|---|
| Core count | 88 Olympus cores (NVIDIA-custom Arm design, Armv9.2 ISA) |
| Threads | 176 |
| IPC improvement | 1.5x over Grace (Neoverse V2-based) |
| Memory | Up to 1.5 TB LPDDR5X via SOCAMM modules |
| Memory bandwidth | 1.2 TB/s |
| I/O | PCIe Gen 6, CXL 3.1 |
| NVLink-C2C | 1.8 TB/s die-to-die (2x Grace's 900 GB/s, 7x PCIe 6.0) |
| Precision support | First CPU with FP8 support |

The Olympus cores are not stock Arm Neoverse designs -- NVIDIA has made custom microarchitectural modifications to optimize for AI-adjacent workloads: reinforcement learning orchestration, agentic reasoning, tool-calling dispatch, and high-throughput data preprocessing. The 1.5x IPC improvement is described by NVIDIA as "a massive generational jump relative to other competing architectures" ([Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-unveils-details-of-new-88-core-vera-cpus-positioned-to-compete-with-amd-and-intel-new-vera-cpu-rack-features-256-liquid-cooled-chips-that-deliver-up-to-a-6x-gain-in-cpu-throughput)).

The NVLink-C2C interconnect at 1.8 TB/s is particularly important: it is the link between the Vera CPU and its paired Rubin GPU within a "Vera Rubin Superchip." This coherent die-to-die link means the CPU and GPU share a unified memory space with dramatically lower latency than PCIe-attached configurations.

For storage engineers, the PCIe Gen 6 and CXL 3.1 support is noteworthy. CXL 3.1 enables memory pooling and sharing across devices, which has implications for how future NVMe-oF and CXL-attached storage tiers can be integrated into the CPU's address space.

### 2.2 Rubin GPU (R100/VR200)

The Rubin GPU is the primary compute engine of the platform. Key specifications:

| Specification | Detail |
|---|---|
| Process | TSMC 3nm |
| Transistor count | 336 billion (1.6x Blackwell's 208B) |
| Die architecture | Multi-chip module: 2 compute dies + 2 I/O dies on 4x reticle CoWoS-L interposer |
| Streaming Multiprocessors | 224 SMs with 6th-gen Tensor Cores |
| Precision support | FP4, FP6, FP8, FP16, BF16, TF32, FP32, FP64 |
| HBM4 memory | 288 GB across 8 stacks |
| HBM4 bandwidth | 22 TB/s per GPU (2.8x Blackwell's 8 TB/s) |
| NVLink 6 bandwidth | 3.6 TB/s per GPU (2x Blackwell) |
| FP4 inference | 50 PFLOPS per GPU (5x Blackwell) |
| FP4 training | 35 PFLOPS per GPU (3.5x Blackwell) |

The 22 TB/s of HBM4 bandwidth per GPU is the single most impactful specification for inference workloads. At this bandwidth, a single R100 can sustain much larger active KV caches during long-context inference without stalling the compute pipeline. The third-generation Transformer Engine adds hardware-accelerated adaptive compression that dynamically adjusts precision across transformer layers using a two-level micro-block scaling scheme for the NVFP4 format ([NVIDIA Technical Blog](https://developer.nvidia.com/blog/inside-the-nvidia-rubin-platform-six-new-chips-one-ai-supercomputer/)).

At the NVL72 rack level, 72 Rubin GPUs provide a combined 20.7 TB of HBM4 capacity and 260 TB/s of NVLink 6 scale-up bandwidth. This is the "hot" tier of the memory hierarchy -- the data that is actively being computed on.

### 2.3 Rubin Ultra GPU (2027)

While not part of the current Vera Rubin Pod shipping in 2026, the Rubin Ultra roadmap is relevant for capacity planning:

| Specification | Detail |
|---|---|
| Die architecture | 4 compute chiplets (2x base Rubin) |
| HBM4e memory | 1 TB across 16 stacks (16 layers of 32Gb DRAM die) |
| HBM4e bandwidth | ~32 TB/s |
| FP4 inference | 100 PFLOPS per GPU |
| TDP | 3.6 kW per GPU |
| Rack design | Kyber rack, 144 GPU packages |
| NVLink generation | NVLink 7 (retains 3.6 TB/s per port, increases GPU count) |

The jump from 288 GB to 1 TB of HBM per GPU will dramatically change the economics of KV cache storage tiering. Models that currently require CMX SSD offload may fit entirely in HBM4e with Rubin Ultra ([Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-demonstrates-rubin-ultra-tray-worlds-1st-ai-gpu-with-1tb-of-hbm4e)).

### 2.4 BlueField-4 DPU

The BlueField-4 is a dual-die package that serves as the infrastructure processor for the pod:

| Specification | Detail |
|---|---|
| CPU | 64-core NVIDIA Grace (Arm-based) |
| Networking | Integrated ConnectX-9, 800 Gb/s throughput |
| Compute improvement | 6x over BlueField-3 |
| Factory scale | Supports AI factories 4x larger than BlueField-3 |
| Key functions | Storage offload, data integrity, encryption, NVMe management, network virtualization |

In the CMX storage racks, BlueField-4 is the engine that manages NVMe SSDs, runs storage services, and handles data integrity and encryption for KV cache data. Each ICMSP (Inference Context Memory Storage Platform) storage enclosure contains 4 BlueField-4 DPUs, each managing 150 TB of NVMe SSD capacity ([NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/)).

For storage engineers, BlueField-4 is the critical component: it is the programmable data path between NVMe SSDs and the GPU fabric. The 64-core Grace CPU inside each BlueField-4 provides substantial compute for inline data services -- compression, deduplication, encryption, and intelligent prefetching of KV cache segments.

### 2.5 ConnectX-9 SuperNIC

| Specification | Detail |
|---|---|
| Throughput | 800 Gb/s per chip |
| Scale-out bandwidth | 1.6 Tb/s per GPU (multiple CX-9 chips packaged together) |
| Protocol | RoCEv2 (RDMA over Converged Ethernet) |
| Optimization | Spectrum-X Ethernet fabric acceleration |

ConnectX-9 provides the scale-out networking for east-west GPU-to-GPU traffic across racks and for north-south traffic to storage. The 1.6 Tb/s per GPU of scale-out bandwidth is the pipe through which CMX storage racks deliver KV cache data to GPUs in remote NVL72 racks ([NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)).

### 2.6 NVLink 6 Switch

| Specification | Detail |
|---|---|
| Ports | 144 NVLink ports |
| Non-blocking switching capacity | 14.4 TB/s per switch chip |
| Per-GPU bandwidth | 3.6 TB/s bidirectional |
| Rack-level bandwidth | 260 TB/s across NVL72 |
| SHARP support | 4x bandwidth efficiency with FP8 in-network reduction |

The NVLink 6 Switch enables all 72 GPUs within an NVL72 rack to communicate in an all-to-all topology at full bandwidth. The SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) capability performs FP8 reductions directly in the switch fabric, reducing the data that must traverse the network for collective operations by up to 4x ([NVIDIA NVLink](https://www.nvidia.com/en-us/data-center/nvlink/)).

### 2.7 Spectrum-6 Ethernet Switch

| Specification | Detail |
|---|---|
| SN6810 switching capacity | 102.4 Tb/s, 128x 800G ports, 2U form factor |
| SN6800 switching capacity | 409.6 Tb/s, 512x 800G ports |
| Architecture | Single-chip design with integrated SerDes |
| Co-packaged optics | Support for silicon photonics-based interconnects |

Spectrum-6 is the scale-out Ethernet fabric that connects the five rack types within the pod. It provides the RDMA transport for CMX storage access, CPU rack communication, and inter-pod networking. The Spectrum-X software stack optimizes RoCE performance with adaptive routing, congestion control, and telemetry specifically tuned for AI traffic patterns ([NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/)).

---

## 3. The Five Rack-Scale Systems

The Vera Rubin Pod is not a homogeneous cluster. It is composed of five distinct rack types, each optimized for a specific role in the agentic AI inference and training pipeline. All five are built on the third-generation NVIDIA MGX modular rack architecture with 100% liquid cooling and cable-free modular tray designs.

### 3.1 DGX NVL72 -- GPU Compute Racks

**Composition:** 72 Rubin GPUs + 36 Vera CPUs (36 Vera Rubin Superchips), interconnected by NVLink 6 switches.

**Purpose:** The primary compute engine for pretraining, post-training, and inference. Each NVL72 rack delivers 3.6 exaflops of FP4 inference, 20.7 TB of HBM4 memory, and 260 TB/s of NVLink 6 scale-up bandwidth.

**Design:** Each superchip (1 Vera CPU + 2 Rubin GPUs) slides into one of 18 compute trays. NVIDIA claims 18x faster assembly and servicing than Blackwell due to the cable-free, modular tray design. This is the first NVIDIA system that is 100% liquid cooled, eliminating fans entirely.

**Power:** Two profiles are available -- Max Q at approximately 190 kW per rack (1.8 kW per GPU) and Max P at approximately 230 kW per rack (2.3 kW per GPU).

In a full 40-rack Vera Rubin Pod, 16 NVL72 racks provide the 1,152 Rubin GPUs that deliver the headline 60 exaflops figure ([NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)).

### 3.2 Groq 3 LPX Racks -- Decode Acceleration

**Composition:** 256 Groq 3 LPUs (Language Processing Units) per rack, built on MGX infrastructure.

**Purpose:** Dedicated decode/inference acceleration for trillion-parameter models and million-token contexts. The LPX architecture is optimized for the autoregressive token-generation phase of inference, which is memory-bandwidth-bound and poorly utilizes traditional GPU compute.

**Significance:** This is the first time NVIDIA has integrated a non-GPU accelerator (Groq's LPU) into its platform. The LPX racks handle the decode phase while NVL72 racks handle prefill and training, creating a heterogeneous inference pipeline that maximizes throughput per watt across both phases ([The Decoder](https://the-decoder.com/gtc-2026-with-groq-3-lpx-nvidia-adds-dedicated-inference-hardware-to-its-platform-for-the-first-time/)).

### 3.3 Vera CPU Rack -- Orchestration and RL

**Composition:** 256 Vera CPUs per rack, liquid cooled, built on MGX.

**Purpose:** Dense CPU compute for reinforcement learning (RL) training loops, agentic reasoning orchestration, tool-calling dispatch, and general-purpose preprocessing. Connected to the pod via Spectrum-X Ethernet.

**Performance:** NVIDIA claims up to 6x CPU throughput compared to the Grace CPU generation. The CPU rack provides the "brain" for agentic AI workflows where a CPU must orchestrate multi-step reasoning chains, invoke external tools, manage state, and dispatch work to GPU and LPU racks ([Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-unveils-details-of-new-88-core-vera-cpus-positioned-to-compete-with-amd-and-intel-new-vera-cpu-rack-features-256-liquid-cooled-chips-that-deliver-up-to-a-6x-gain-in-cpu-throughput)).

### 3.4 CMX / BlueField-4 STX Racks -- Context Memory Storage

**Composition:** BlueField-4 DPUs managing NVMe SSD arrays. Each ICMSP enclosure contains 4 BlueField-4 DPUs, each with 150 TB of NVMe SSD capacity behind it.

**Purpose:** This is the rack type of greatest interest to storage engineers. CMX establishes a new "G3.5" storage tier in NVIDIA's memory hierarchy:

- **G3 (Local):** HBM4 on the GPU die -- 288 GB per GPU, 22 TB/s
- **G3.5 (CMX):** Ethernet-attached NVMe flash, optimized for KV cache -- petabytes of capacity, accessed via RDMA
- **G4 (Shared):** Traditional shared enterprise storage (parallel file systems, object stores)

CMX acts as the "agentic long-term memory" of the pod. It stores KV cache data that is too large to fit in HBM but must be accessed at much lower latency than traditional storage. The KV cache for a multi-turn agentic conversation with million-token context can easily exceed 100 GB per session; multiplied across thousands of concurrent sessions, this far exceeds HBM capacity.

**Performance claims:** CMX delivers 5x higher tokens-per-second (TPS) and 5x better power efficiency than serving the same workloads from traditional storage. The BlueField-4 DPUs run inline storage services -- data integrity checks, encryption, and intelligent pre-staging of KV cache segments back into GPU HBM before they are needed by the decode engine.

**Networking:** Spectrum-X Ethernet provides the RDMA fabric (RoCEv2) for low-latency access between GPU racks and CMX storage racks. The entire data path from SSD to GPU is designed to avoid CPU involvement on the compute side, using BlueField-4's DMA engines to move data directly ([NVIDIA CMX](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/); [Blocks and Files](https://www.blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/4090541)).

### 3.5 Spectrum-6 SPX Racks -- Networking Fabric

**Composition:** Spectrum-6 Ethernet switches in spine/leaf topology.

**Purpose:** Connects all rack types within the pod into a single, unified supercomputer. Provides the scale-out Ethernet fabric for GPU-to-GPU traffic across NVL72 racks (supplementing NVLink within racks), CPU rack connectivity, and storage rack access.

**Specifications:** The SPX racks use Spectrum-6 SN6810 (102.4 Tb/s) and SN6800 (409.6 Tb/s) switches with 800G port speeds. The Spectrum-X software stack provides AI-optimized adaptive routing, congestion control, and flow scheduling ([NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/)).

---

## 4. Memory and Bandwidth Hierarchy

For storage engineers, the most important aspect of the Vera Rubin Pod is its multi-tier memory and bandwidth hierarchy. Each tier has distinct capacity, bandwidth, and latency characteristics that determine what data lives where.

### 4.1 Tier Overview

```
Tier        Medium          Capacity (per GPU)   Bandwidth (per GPU)   Latency       Scope
──────────  ──────────────  ──────────────────   ───────────────────   ────────────  ──────────────
HBM4        On-package      288 GB               22 TB/s               ~ns           GPU-local
NVLink 6    Copper/optical  20.7 TB (rack)       3.6 TB/s              ~us           Intra-rack
Spectrum-X  Ethernet RDMA   Pod-wide             1.6 Tb/s (200 GB/s)  ~10s of us    Inter-rack
CMX (SSD)   NVMe flash      150+ TB per BF4      SSD-limited           ~100s of us   Pod-attached
G4 Storage  PFS/Object      Unlimited            Network-limited       ~ms           Data center
```

### 4.2 HBM4: The Hot Tier (22 TB/s per GPU)

HBM4 is the only memory tier that can feed the Tensor Cores at full throughput. At 22 TB/s per GPU, the Rubin GPU has 2.8x the memory bandwidth of Blackwell (8 TB/s) and 6.6x that of Hopper H100 (3.35 TB/s). The 288 GB capacity per GPU (20.7 TB per NVL72 rack) determines the maximum "working set" for active inference without any offload.

For large language models with long contexts, the KV cache grows linearly with sequence length and batch size. A single inference session with 1M tokens of context on a large MoE model can consume 50-100+ GB of KV cache. At scale, the KV cache dominates HBM consumption, often exceeding the model weights themselves.

### 4.3 NVLink 6: The Scale-Up Fabric (260 TB/s per rack)

NVLink 6 provides 3.6 TB/s of bidirectional bandwidth per GPU -- 14x faster than PCIe Gen 6 and 2x faster than NVLink 5 (Blackwell). Within the NVL72 rack, all 72 GPUs communicate through the NVLink 6 Switch fabric at an aggregate 260 TB/s.

This bandwidth enables the 72 GPUs to operate as a single logical accelerator with 20.7 TB of unified HBM4 memory. For inference, this means KV cache data can be distributed across all 72 GPUs and accessed by any GPU at NVLink speeds, effectively creating a 20.7 TB "shared" HBM pool within the rack.

The SHARP in-network reduction capability of the NVLink 6 Switch is important for training: all-reduce operations for gradient synchronization happen partially within the switch fabric itself, reducing the data volume that traverses the links.

### 4.4 Spectrum-X / RDMA: The Scale-Out Fabric (1.6 Tb/s per GPU)

ConnectX-9 SuperNICs provide 1.6 Tb/s (200 GB/s) of scale-out bandwidth per GPU over the Spectrum-X Ethernet fabric. This is the transport layer for:

- **Inter-rack GPU communication:** When a workload spans multiple NVL72 racks, Spectrum-X carries the cross-rack traffic.
- **CMX storage access:** KV cache reads and writes between GPU racks and CMX storage racks traverse this fabric.
- **CPU rack communication:** Agentic orchestration traffic between Vera CPU racks and GPU racks.

The bandwidth gap between NVLink 6 (3.6 TB/s) and Spectrum-X (200 GB/s) is approximately 18x. This asymmetry is by design: NVLink handles the latency-sensitive, high-bandwidth compute communication within a rack, while Spectrum-X handles the bulk data movement and storage access across the pod.

### 4.5 CMX: The Warm Storage Tier (NVMe Flash)

CMX is the new tier that storage engineers need to understand most deeply. It sits between HBM and traditional shared storage, purpose-built for one workload: KV cache for agentic inference.

**Architecture:** BlueField-4 DPUs manage arrays of NVMe SSDs. Each BlueField-4 manages 150 TB of flash. The DPU handles:

- RDMA target services (exposing SSDs to the GPU fabric as RDMA-accessible memory)
- Data integrity and checksums
- Encryption at rest and in transit
- Intelligent prefetching (pre-staging KV cache segments into GPU HBM before they are needed)
- Garbage collection and wear leveling coordination

**Access pattern:** KV cache offload is a write-heavy workload during the prefill phase (when the model processes input tokens and generates KV pairs) and a read-heavy workload during the decode phase (when the model generates output tokens and must read back previously computed KV pairs). The access pattern is largely sequential within a session but random across sessions.

**Why SSDs and not DRAM?** The economics are straightforward. A million-token KV cache for a single session on a large model can be 50-100 GB. Serving thousands of concurrent agentic sessions requires petabytes of KV cache. At current pricing, this is economically infeasible in DRAM or HBM but viable in high-performance NVMe flash. The latency penalty (microseconds for RDMA + SSD vs. nanoseconds for HBM) is acceptable for KV cache because the decode phase is inherently sequential -- each token depends on the previous one, so the pipeline has natural stalls where prefetching can hide SSD latency.

**NVIDIA's positioning:** NVIDIA describes CMX as establishing "a new G3.5 layer, an Ethernet-attached flash tier optimized specifically for KV cache. This tier acts as the agentic long-term memory of the AI infrastructure pod that is large enough to hold shared, evolving context for many agents simultaneously, but also close enough for the context to be pre-staged frequently back into GPU and host memory without stalling decode" ([Blocks and Files](https://blocksandfiles.com/2026/01/06/nvidia-standardizes-gpu-cluster-kv-cache-offload-to-nvme-ssds/)).

### 4.6 G4: Traditional Shared Storage

The outermost tier is conventional data center storage -- parallel file systems (Lustre, GPFS/Spectrum Scale, WEKA, VAST), object stores (S3-compatible), and network-attached storage. This tier handles:

- Training dataset staging
- Checkpoint storage
- Model weight distribution
- Log and telemetry collection

The bandwidth to G4 storage is limited by the data center network, typically 100-400 Gb/s per node. This tier is not in the critical inference data path for Vera Rubin; CMX absorbs the latency-sensitive KV cache workload that would otherwise require expensive high-performance shared storage.

---

## 5. Comparison to Previous Generations

### 5.1 Vera Rubin vs. Blackwell: Key Metrics

| Metric | Blackwell (GB200 NVL72) | Vera Rubin (VR200 NVL72) | Improvement |
|---|---|---|---|
| GPU process node | TSMC 4nm | TSMC 3nm | 1 node |
| Transistors per GPU | 208 billion | 336 billion | 1.6x |
| HBM type | HBM3e | HBM4 | 1 generation |
| HBM capacity per GPU | 192 GB | 288 GB | 1.5x |
| HBM bandwidth per GPU | 8 TB/s | 22 TB/s | 2.75x |
| NVLink bandwidth per GPU | 1.8 TB/s | 3.6 TB/s | 2x |
| NVLink generation | NVLink 5 | NVLink 6 | 1 generation |
| Rack NVLink bandwidth | 130 TB/s | 260 TB/s | 2x |
| FP4 inference per GPU | ~10 PFLOPS | 50 PFLOPS | 5x |
| FP4 training per GPU | ~10 PFLOPS | 35 PFLOPS | 3.5x |
| Rack HBM capacity | ~14 TB | 20.7 TB | 1.5x |
| CPU | Grace (72 Neoverse V2 cores) | Vera (88 Olympus cores) | +22% cores, 1.5x IPC |
| CPU memory | LPDDR5X | LPDDR5X | Same generation |
| CPU-GPU link | NVLink-C2C 900 GB/s | NVLink-C2C 1.8 TB/s | 2x |
| DPU | BlueField-3 | BlueField-4 | 6x compute |
| NIC | ConnectX-7 (400G) | ConnectX-9 (800G) | 2x |
| Scale-out per GPU | 400 Gb/s | 1.6 Tb/s | 4x |
| Cooling | Hybrid air/liquid | 100% liquid | Full transition |
| Inference cost per token | Baseline | Up to 10x reduction | 10x |
| GPUs needed for MoE training | Baseline | 4x fewer | 4x |
| Assembly/service time | Baseline | 18x faster | 18x |

Sources: [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform); [Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/nvidia-launches-vera-rubin-nvl72-ai-supercomputer-at-ces-promises-up-to-5x-greater-inference-performance-and-10x-lower-cost-per-token-than-blackwell-coming-2h-2026); [WCCFTech](https://wccftech.com/nvidia-rubin-most-advanced-ai-platform-50-pflops-vera-cpu-5x-uplift-vs-blackwell/)

### 5.2 Architectural Differences

Beyond the raw numbers, several architectural shifts distinguish Vera Rubin from Blackwell:

**Pod-level design rather than server-level design.** Blackwell was primarily designed as a rack-scale system (NVL72) that could be aggregated into SuperPODs. Vera Rubin is designed from the start as a 40-rack pod with five heterogeneous rack types, each optimized for a specific function. The pod is the unit of deployment, not the rack.

**Heterogeneous compute.** The addition of Groq 3 LPX racks for decode and dedicated Vera CPU racks for orchestration means the pod has three distinct compute engines (GPU, LPU, CPU), each handling the phase of the inference pipeline where it is most efficient. Blackwell used GPUs for everything.

**First-class storage tier.** CMX/STX racks did not exist in the Blackwell generation. KV cache offload to SSDs was a software-level optimization; with Vera Rubin, it is a hardware-defined tier with dedicated silicon (BlueField-4), dedicated racks, and a purpose-built RDMA data path.

**100% liquid cooling.** Blackwell supported hybrid air/liquid configurations. Vera Rubin is exclusively liquid cooled, enabling higher power density per rack and "much less water" consumption than traditional evaporative cooling approaches ([CNBC](https://www.cnbc.com/2026/02/25/first-look-at-nvidias-ai-system-vera-rubin-and-how-it-beats-blackwell.html)).

**Cable-free modular tray design.** Compute trays slide in and out without cable management. NVIDIA claims 18x faster assembly and servicing, which directly impacts data center operational costs and mean-time-to-repair.

---

## 6. Key Takeaways for Storage Engineers

### 6.1 KV Cache Is Now a Storage Problem

The single most important takeaway from the Vera Rubin Pod architecture is that NVIDIA has formally declared KV cache management to be a storage infrastructure problem, not merely a GPU memory management problem. The CMX platform with BlueField-4-managed NVMe SSDs creates a new tier that storage teams must plan for, provision, and operate.

The implications are concrete:

- **SSD procurement:** CMX racks require high-endurance, high-IOPS NVMe SSDs optimized for the mixed read/write pattern of KV cache offload. Standard data center read-heavy SSDs may not be appropriate. NVIDIA's partner ecosystem (Solidigm, Samsung, SK hynix, Micron) will likely offer CMX-qualified drives.
- **Capacity planning:** KV cache size scales with context length, model size, and concurrent sessions. For million-token contexts on 1T+ parameter MoE models, plan for 50-100 GB of KV cache per active session. A pod serving 10,000 concurrent sessions needs 500 TB to 1 PB of CMX capacity.
- **Endurance planning:** KV cache is written during prefill and read during decode. With multi-turn agentic conversations, KV cache is frequently updated. The write amplification and endurance requirements will depend heavily on the inference workload mix.
- **Network fabric:** CMX access runs over Spectrum-X Ethernet with RDMA. Storage engineers must plan the Ethernet fabric to provide sufficient bandwidth between NVL72 GPU racks and CMX storage racks, with appropriate QoS to avoid interference between storage traffic and inter-rack GPU traffic.

### 6.2 The G3.5 Tier Changes the Storage Hierarchy

Traditional AI infrastructure had two storage tiers relevant to inference: HBM (fast, small, expensive) and shared storage (slow, large, cheap). CMX introduces a middle tier that changes the optimization calculus:

- Models that previously required more GPUs (to fit KV cache in HBM) can now use fewer GPUs plus CMX storage, potentially reducing total cost of ownership.
- NVIDIA's claim of 4x fewer GPUs needed for MoE training is partially enabled by offloading intermediate state to CMX.
- The 5x TPS improvement from CMX vs. traditional storage suggests that previous SSD-based KV cache approaches (without BlueField-4 acceleration and RDMA optimization) were leaving significant performance on the table.

### 6.3 BlueField-4 as a Storage Controller

BlueField-4 is not just a network adapter -- it is a full storage controller with a 64-core Grace CPU. This is more compute than many standalone storage arrays ship with. Storage engineers should expect BlueField-4 to run sophisticated storage services: inline compression, deduplication, encryption, wear-leveling coordination, and predictive prefetching. The programmability of BlueField-4 means these services will evolve with software updates over the life of the hardware.

### 6.4 Plan for Power and Cooling

A full Vera Rubin Pod at 40 racks will consume multiple megawatts. The CMX storage racks add to this power budget. Storage engineers working on facility planning must account for:

- 100% liquid cooling infrastructure for all rack types
- Power delivery for both compute racks (190-230 kW each) and storage racks
- The elimination of air cooling simplifies HVAC but requires robust liquid cooling distribution

### 6.5 Rubin Ultra Will Shift the Tier Boundaries

When Rubin Ultra ships in 2027 with 1 TB of HBM4e per GPU (72 TB per rack), many workloads that currently spill to CMX will fit entirely in HBM. However, model sizes and context lengths are growing faster than HBM capacity, so CMX will likely remain relevant even with Rubin Ultra. Storage engineers should plan CMX deployments as a durable part of the infrastructure, not a stopgap.

### 6.6 Spectrum-X Ethernet Is the Storage Network

Unlike InfiniBand-era HPC storage, CMX runs on Ethernet with RDMA (RoCEv2). This means storage engineers can leverage existing Ethernet operational expertise, tooling, and monitoring. However, the performance requirements are extreme: 800 Gb/s links, ultra-low latency, and lossless Ethernet behavior. The Spectrum-X software stack provides the AI-specific optimizations (adaptive routing, congestion control, flow scheduling) that make this feasible, but it requires Spectrum-6 switches -- commodity Ethernet switches will not deliver the required performance.

---

## Sources

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
