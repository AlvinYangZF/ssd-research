# Interconnects: NVLink 6 & Spectrum-X

**A technical architecture analysis for storage engineers evaluating AI inference infrastructure**

## Abstract

The NVIDIA Vera Rubin platform introduces two fundamentally distinct interconnect tiers — NVLink 6 for GPU-to-GPU communication and Spectrum-X Ethernet for rack-to-rack and GPU-to-storage fabric — that together define the data movement capabilities of modern AI factories. For storage engineers, understanding these interconnects is not optional: they dictate the achievable throughput and latency for every KV cache retrieval, every model checkpoint load, and every context memory access in the CMX storage architecture. This report dissects both interconnect technologies, traces the complete data path from GPU HBM through NVLink, PCIe Gen6, and Spectrum-X Ethernet down to NVMe SSDs, and identifies the bandwidth bottlenecks that storage system design must account for. All specifications reflect the Vera Rubin platform as announced through GTC 2026.

---

## 1. Introduction: Why Interconnects Are the Backbone of Disaggregated AI Infrastructure

The shift from monolithic GPU servers to disaggregated, rack-scale AI systems has transformed interconnects from a supporting component into the primary architectural constraint. In the Vera Rubin pod — a 40-rack AI supercomputer integrating 1,152 Rubin GPUs, Vera CPUs, BlueField-4 DPUs, and NVMe storage — data must traverse multiple fabric tiers to reach its destination. A single KV cache lookup during agentic AI inference may originate in a GPU's execution pipeline, cross an NVLink domain to reach the associated Vera CPU, traverse PCIe Gen6 to a ConnectX-9 SuperNIC, travel across a Spectrum-X Ethernet fabric to a BlueField-4 STX storage node, and finally access NVMe flash. Each hop introduces its own bandwidth ceiling and latency penalty.

For storage engineers, two questions dominate system design:

1. **What is the end-to-end bandwidth from GPU to NVMe SSD?** The answer determines how many concurrent KV cache retrievals a CMX storage rack can serve before becoming a bottleneck.
2. **What is the end-to-end latency budget?** Agentic AI inference is latency-sensitive — each token generation step may require context retrieval from flash, and that retrieval must complete within the model's inter-token deadline.

The answers lie in understanding NVLink 6 and Spectrum-X not as isolated technologies, but as complementary tiers in a hierarchical data movement architecture.

---

## 2. NVLink 6: The GPU-to-GPU Fabric

### 2.1 Specifications and Generational Leap

NVLink 6, introduced with the Rubin GPU architecture, doubles per-GPU interconnect bandwidth compared to NVLink 5 on Blackwell ([NVIDIA NVLink](https://www.nvidia.com/en-us/data-center/nvlink/)):

| Parameter | NVLink 5 (Blackwell) | NVLink 6 (Rubin) |
|-----------|---------------------|-------------------|
| Per-GPU bandwidth | 1,800 GB/s | 3,600 GB/s |
| NVLink ports per GPU | 18 | 36 |
| Bandwidth per link | 100 GB/s | ~100 GB/s (bidirectional) |
| Rack-level aggregate (NVL72) | 130 TB/s | 260 TB/s |
| vs. PCIe Gen6 | ~7x | ~14x |

Each Rubin R200 GPU supports 36 NVLink 6 connections, providing 3.6 TB/s of bidirectional bandwidth — over 14 times the bandwidth of a PCIe Gen6 x16 link ([NVIDIA Vera Rubin NVL72](https://www.nvidia.com/en-us/data-center/vera-rubin-nvl72/)). This bandwidth is consumed primarily by all-to-all collective operations during mixture-of-experts (MoE) inference and gradient synchronization during training.

### 2.2 NVLink 6 Switch Architecture

The NVLink 6 Switch is one of the seven silicon chips in the Vera Rubin platform. Unlike Ethernet or InfiniBand switches that operate at the rack-to-rack level, NVLink switches create a flat, all-to-all GPU communication domain within a single rack.

Key switch characteristics ([Decoding Nvidia's Rubin Networking Math](https://www.wheelersnetwork.com/2025/11/decoding-nvidias-rubin-networking-math.html)):

- **Switch tray configuration:** Each NVLink switch tray in the VR200 NVL72 (also referred to as NVL144, counting individual GPU dies) contains multiple NVLink 6 ASICs. The Rubin generation doubles bandwidth by using twice the switch ASICs compared to Blackwell's NVLink 5.
- **Non-blocking fabric:** All 72 GPUs within an NVL72 rack are connected in a fully non-blocking, all-to-all topology. Any GPU can communicate with any other GPU at full line rate without contention.
- **In-network compute:** NVLink switches support NVIDIA SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) with FP8 precision, enabling collective operations (allreduce, allgather) to be partially executed within the switch fabric itself. Each switch tray delivers 14.4 TFLOPS of FP8 in-network compute, reducing the volume of data that must traverse the full fabric.
- **Aggregate switching capacity:** The NVL72's switch trays collectively provide 260 TB/s of bisection bandwidth across the 72-GPU domain.

### 2.3 Multi-Rack Scalability: DGX SuperPOD

A single NVL72 rack is the fundamental compute building block, but production AI factories require multi-rack scale. NVIDIA's DGX SuperPOD with Vera Rubin NVL72 unifies eight NVL72 systems — 576 Rubin GPUs total — delivering up to 28.8 exaflops of FP4 performance and 600 TB of aggregate HBM capacity ([NVIDIA DGX SuperPOD](https://blogs.nvidia.com/blog/dgx-superpod-rubin/)).

Inter-rack communication between NVL72 systems does not use NVLink. Instead, it relies on Spectrum-X Ethernet or Quantum-X800 InfiniBand as the scale-out fabric. This architectural boundary — NVLink within the rack, Ethernet/InfiniBand between racks — is critical for storage engineers to understand: **storage traffic always traverses the Ethernet/InfiniBand scale-out fabric, never the NVLink domain directly.**

### 2.4 Comparison with NVLink 5 (Blackwell)

The generational improvement from NVLink 5 to NVLink 6 is primarily in raw bandwidth rather than architectural redesign ([NVLink 6 Becomes the Backbone of Rubin Rack-Scale AI Architecture](https://convergedigest.com/nvlink-6-becomes-the-backbone-of-rubin-rack-scale-ai-architecture/)):

- **2x per-GPU bandwidth:** 3,600 GB/s vs. 1,800 GB/s. Achieved through doubling NVLink ports per GPU (36 vs. 18) and doubling switch ASICs per tray.
- **Same logical topology:** Both NVLink 5 and NVLink 6 provide all-to-all connectivity within a 72-GPU NVL72 domain. The switch architecture scales linearly.
- **Improved MoE throughput:** For all-to-all operations critical to MoE routing, NVLink 6 delivers up to 2x higher throughput compared to NVLink 5, directly benefiting inference workloads that route tokens across expert subnetworks.
- **Same rack boundary:** Neither NVLink 5 nor NVLink 6 extends beyond a single rack natively. Multi-rack communication remains the domain of Ethernet or InfiniBand.

### 2.5 NVLink's Role in the CMX Data Path

While NVLink itself does not carry storage traffic, it plays an indirect but important role in CMX context memory access. In the Vera Rubin architecture, each Rubin GPU is paired with a Vera CPU via a high-bandwidth coherent interconnect (the NVLink-C2C chip-to-chip link within the Grace-like superchip packaging). When a GPU needs to access KV cache stored on a remote CMX storage node:

1. The GPU issues a memory access that traverses the local NVLink-C2C link to the paired Vera CPU.
2. The Vera CPU's networking stack initiates an RDMA read over Spectrum-X Ethernet via the ConnectX-9 SuperNIC.
3. The return data path reverses this flow.

The NVLink-C2C link between GPU and CPU within a compute tray operates at high bandwidth (the Vera CPU supports up to 1.2 TB/s of memory bandwidth), but this hop is distinct from the inter-GPU NVLink 6 fabric discussed above. Storage engineers should not conflate the two.

---

## 3. Spectrum-X Ethernet: The Scale-Out and Storage Fabric

### 3.1 Architecture Overview

NVIDIA Spectrum-X is a purpose-built Ethernet networking platform designed for AI workloads, coupling two co-designed components ([NVIDIA Spectrum-X](https://www.nvidia.com/en-us/networking/spectrumx/)):

1. **Spectrum-6 Ethernet Switch ASIC** — a 102.4 Tb/s single-chip switch with co-packaged optics.
2. **ConnectX-9 SuperNIC** — an 800GbE network adapter with hardware RDMA offload, PCIe Gen6 host interface, and AI-specific acceleration.

Together, these components deliver what NVIDIA calls "InfiniBand-class performance over Ethernet" — specifically, high-throughput RDMA over Converged Ethernet (RoCE) with adaptive routing and congestion control that approaches the deterministic behavior traditionally associated with InfiniBand.

### 3.2 Spectrum-6 Switch ASIC

The Spectrum-6 is the latest generation of NVIDIA's Ethernet switch silicon, and it represents a significant departure from conventional merchant switch ASICs ([Spectrum-6 Ethernet Switch Deep Dive](https://www.naddod.com/ai-insights/spectrum-6-ethernet-switch-deep-dive-sn6810-102-4t-and-sn6800-409-6t-switch)):

| Parameter | Specification |
|-----------|--------------|
| Aggregate switching capacity | 102.4 Tb/s |
| SerDes rate | 224G per lane |
| Port configurations | 512 x 200G or 128 x 800G |
| Optics | Co-packaged silicon photonics (32 x 1.6 Tb/s engines) |
| Power efficiency vs. pluggable | ~5x better optical power efficiency |
| Resiliency vs. pluggable | ~10x higher optical resiliency |

The co-packaged optics (CPO) design is architecturally significant. By integrating 32 silicon photonics optical engines directly into the switch package using Micro Ring Modulator technology, Spectrum-6 eliminates the signal integrity losses that plague conventional designs where 224G SerDes signals must traverse long PCB traces to front-panel pluggable transceivers. This is not merely an efficiency improvement — at 224G per lane, traditional electrical interconnects between the switch ASIC and pluggable optics modules introduce enough signal degradation to require power-hungry retimers or reduce achievable link distances.

The SPX (Spectrum Platform for X-scale) rack in the Vera Rubin POD houses these Spectrum-6 switches, providing the spine of the scale-out Ethernet fabric that interconnects NVL72 compute racks, STX storage racks, and Vera CPU racks ([NVIDIA Vera Rubin POD](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)).

### 3.3 ConnectX-9 SuperNIC

The ConnectX-9 is NVIDIA's latest network adapter, purpose-built for AI factory deployments. It serves as the endpoint for Spectrum-X Ethernet in both compute nodes (NVL72) and storage nodes (STX) ([NVIDIA ConnectX-9 SuperNIC](https://docs.nvidia.com/networking/display/connectx9supernic)):

| Parameter | Specification |
|-----------|--------------|
| Ethernet speed | 1 x 800GbE (single port, no aggregation) |
| Host interface | PCIe Gen6 x16 (dual x16 via Socket Direct) |
| RDMA | RoCEv2 with hardware offload |
| Dual-socket support | Socket Direct — split 32-lane PCIe bus across two x16 interfaces |
| Security | Post-Quantum Cryptography (PQC), CNSA 2.0 compliant |
| Form factor | OSFP |

The 800GbE single-port capability is notable: previous-generation SuperNICs required multi-link aggregation to reach comparable throughput. The ConnectX-9 achieves 800 Gb/s (100 GB/s) on a single optical lane group, simplifying cabling and reducing switch port consumption.

The PCIe Gen6 x16 host interface provides approximately 128 GB/s of bidirectional bandwidth (64 GT/s per lane x 16 lanes x 2 bytes per transfer at Gen6 encoding). This means the ConnectX-9's 800GbE line rate (100 GB/s) is comfortably within the PCIe Gen6 x16 envelope, avoiding a host-side bottleneck that plagued earlier generations where NIC line rate could exceed PCIe bandwidth.

### 3.4 RoCE: RDMA over Converged Ethernet

Spectrum-X's performance advantage over commodity Ethernet stems from its RoCE implementation, which goes well beyond the basic RoCEv2 standard. The platform implements three key enhancements ([Turbocharging AI Workloads with Spectrum-X](https://developer.nvidia.com/blog/turbocharging-ai-workloads-with-nvidia-spectrum-x-networking-platform/)):

**RoCE Adaptive Routing:** Traditional Ethernet uses static ECMP (Equal-Cost Multi-Path) hashing to distribute flows across available paths. This creates persistent hot spots when hash collisions concentrate multiple large flows on the same link. Spectrum-X implements fine-grained, per-packet adaptive routing at the switch level. The Spectrum-6 switch dynamically selects the least-congested path for each packet, achieving near-optimal load balancing across all available fabric paths. This is functionally equivalent to InfiniBand's adaptive routing capability — a feature that standard Ethernet switches lack.

**RoCE Congestion Control:** Spectrum-X uses a telemetry-based congestion control mechanism where the switch hardware monitors queue depths and link utilization in real time, then signals the SuperNIC to adjust sender injection rates. This hardware-to-hardware feedback loop operates at microsecond granularity, far faster than software-based congestion control schemes like DCQCN that rely on ECN marking and end-to-end feedback.

**RoCE Performance Isolation:** In multi-tenant or mixed-workload environments, Spectrum-X provides traffic isolation that prevents one flow from degrading another. This is critical in AI factories where storage traffic (CMX KV cache retrievals) shares the Ethernet fabric with inter-rack GPU communication traffic.

The combined effect of these enhancements is substantial: NVIDIA reports up to 95% effective bandwidth utilization at scale and under load, compared to 50-70% typical for commodity RoCE fabrics. In storage-specific testing using the Israel-1 supercomputer, Spectrum-X improved read bandwidth by up to 48% and write bandwidth by up to 41% compared to standard Ethernet ([Accelerating AI Storage with Spectrum-X](https://developer.nvidia.com/blog/accelerating-ai-storage-by-up-to-48-with-nvidia-spectrum-x-networking-platform-and-partners/)).

### 3.5 Spectrum-XGS: Giga-Scale Expansion

For deployments that span multiple data center buildings or campuses, NVIDIA introduced Spectrum-XGS — a scale-across technology that unifies distributed data centers into a single logical AI fabric ([NVIDIA Spectrum-XGS](https://nvidianews.nvidia.com/news/nvidia-introduces-spectrum-xgs-ethernet-to-connect-distributed-data-centers-into-giga-scale-ai-super-factories)). While less immediately relevant to single-pod storage architecture, Spectrum-XGS is significant for storage engineers designing geo-distributed context memory pools where KV cache data may need to be accessible across physical sites.

---

## 4. CMX Data Path Analysis: GPU to NVMe SSD

This section traces the complete data path for a KV cache retrieval from a Rubin GPU to an NVMe SSD in a CMX storage node, identifying the bandwidth ceiling and approximate latency contribution at each hop. This is the analysis most directly relevant to storage engineers.

### 4.1 The Full Path

```
Rubin GPU (HBM4)
  │
  │  NVLink-C2C (chip-to-chip coherent link)
  │  ~900 GB/s per direction
  ▼
Vera CPU
  │
  │  PCIe Gen6 x16
  │  ~128 GB/s bidirectional (64 GB/s per direction)
  ▼
ConnectX-9 SuperNIC
  │
  │  Spectrum-X Ethernet (800GbE RoCE)
  │  100 GB/s line rate per port
  ▼
Spectrum-6 Switch (SPX rack)
  │
  │  Spectrum-X Ethernet (800GbE RoCE)
  │  100 GB/s line rate per port
  ▼
BlueField-4 DPU (STX storage node)
  │
  │  PCIe Gen6 to NVMe controller
  │  ~128 GB/s per PCIe Gen6 x16 root complex
  ▼
NVMe SSD (e.g., Micron 9650)
  ~28 GB/s sequential read per drive
```

### 4.2 Bandwidth at Each Hop

**Hop 1: GPU to Vera CPU (NVLink-C2C)**
The Rubin GPU and Vera CPU are connected via a chip-to-chip coherent link within the compute tray. The Vera CPU supports up to 1.2 TB/s of memory bandwidth with full SOCAMM2 configuration. The C2C link bandwidth between GPU and CPU is not the bottleneck for storage access — it is sized for the much higher bandwidth demands of GPU memory access patterns.

**Bottleneck assessment:** Not a constraint for storage traffic. The C2C link has ample headroom.

**Hop 2: Vera CPU to ConnectX-9 SuperNIC (PCIe Gen6 x16)**
PCIe Gen6 at x16 width delivers approximately 128 GB/s bidirectional (64 GB/s in each direction). The ConnectX-9's 800GbE port runs at 100 GB/s line rate. Since 100 GB/s > 64 GB/s per direction, a single PCIe Gen6 x16 lane cannot fully saturate the NIC in one direction. However, the ConnectX-9's Socket Direct feature splits the 32-lane PCIe bus into two x16 interfaces, potentially providing 128 GB/s per direction in dual-socket configurations.

**Bottleneck assessment:** In single-socket configurations, PCIe Gen6 x16 (64 GB/s per direction) is tighter than the NIC's 100 GB/s line rate. In practice, RDMA operations are bidirectional (commands go out, data comes back), so the 128 GB/s aggregate is adequate for most access patterns. This hop is a moderate constraint.

**Hop 3: ConnectX-9 to Spectrum-6 Switch (800GbE Ethernet)**
The 800GbE link provides 100 GB/s of raw bandwidth. With RoCE protocol overhead (headers, CRC, inter-packet gaps), effective throughput is approximately 95-97 GB/s for large RDMA transfers. For small, random KV cache reads (4KB-64KB), per-message overhead reduces effective throughput significantly — the IOPS ceiling of the NIC becomes the limiting factor rather than raw bandwidth.

**Bottleneck assessment:** For large sequential KV cache transfers (e.g., prefilling a long context), the 800GbE link is adequate. For small random accesses, RDMA message rate and NIC IOPS become the constraint.

**Hop 4: Spectrum-6 Switch Forwarding**
The Spectrum-6 switch operates at 102.4 Tb/s aggregate capacity with cut-through forwarding. Per-port latency through the switch is in the low hundreds of nanoseconds range. The switch itself is not a bandwidth bottleneck for any individual flow — its capacity is designed to handle hundreds of simultaneous 800GbE flows.

**Bottleneck assessment:** Not a constraint. Switch-level congestion is managed by adaptive routing and congestion control.

**Hop 5: Spectrum-6 Switch to BlueField-4 DPU (800GbE Ethernet)**
Symmetric with Hop 3. The BlueField-4 DPU in the STX storage node terminates the RoCE connection and translates it into local NVMe operations.

**Bottleneck assessment:** Same as Hop 3.

**Hop 6: BlueField-4 to NVMe SSD (PCIe Gen6)**
The BlueField-4 manages NVMe SSDs directly, handling NVMe-oF termination, data integrity (CRC), and encryption at line rate. PCIe Gen6 SSDs like the Micron 9650 deliver up to 28 GB/s sequential read throughput per drive. A single STX storage node aggregates multiple SSDs behind the BlueField-4's PCIe root complex.

**Bottleneck assessment:** Individual SSD bandwidth (28 GB/s) is the narrowest point in the entire path. A CMX storage node must stripe across many SSDs in parallel to saturate the upstream 800GbE link. At 100 GB/s effective network bandwidth, approximately 4 SSDs reading in parallel at peak rate are needed to fill the pipe. In practice, with random read patterns typical of KV cache access, more SSDs are needed to compensate for reduced per-drive throughput at high queue depths.

### 4.3 Latency Budget for KV Cache Retrieval

Estimating the end-to-end latency for a single KV cache read from GPU to NVMe and back:

| Hop | Component | Estimated Latency |
|-----|-----------|-------------------|
| 1 | GPU to Vera CPU (C2C) | < 100 ns |
| 2 | CPU to ConnectX-9 (PCIe + driver) | 500 ns - 1 us |
| 3 | ConnectX-9 RDMA processing | 1 - 2 us |
| 4 | Network transit (switch + cable) | 1 - 3 us |
| 5 | BlueField-4 RDMA termination | 1 - 2 us |
| 6 | NVMe SSD access (4KB random read) | 50 - 100 us |
| 7 | Return path (hops 5-1 reverse) | 3 - 6 us |
| **Total** | **End-to-end** | **~60 - 120 us** |

The NVMe SSD access latency dominates the budget at 50-100 microseconds for a 4KB random read on current-generation QLC or TLC flash. All networking and interconnect hops combined contribute roughly 10-15 microseconds — less than 20% of the total. This has a critical implication for storage engineers: **optimizing the network fabric beyond a certain point yields diminishing returns; the SSD media latency is the dominant factor.** Investments in SSD-level optimization (higher queue depths, intelligent prefetching, SLC caching for hot KV entries, computational storage for decompression) deliver more latency reduction than further network tuning.

### 4.4 RDMA Zero-Copy Access

A key performance feature of the CMX data path is RDMA zero-copy operation. In a traditional storage access:

1. Data is read from SSD into DPU memory.
2. Data is copied to a network buffer.
3. Data is transmitted over the network.
4. Data arrives at the host NIC buffer.
5. Data is copied to application memory.

With RDMA zero-copy over Spectrum-X:

1. Data is read from SSD into BlueField-4's registered memory region.
2. The ConnectX-9 on the requesting side issues an RDMA READ that directly transfers data from the BlueField-4's memory into the GPU-accessible memory region — bypassing the Vera CPU's involvement in the data path entirely.
3. The GPU accesses the data from the registered memory region.

This eliminates multiple memory copies and CPU interrupts, reducing both latency and CPU overhead. The BlueField-4's KV I/O plane software is specifically designed to keep hot KV cache entries in DRAM for RDMA serving, falling back to NVMe flash only for cache misses. This tiered access pattern means that frequently accessed context memory can be served at DRAM speed (~1-5 us round-trip) rather than flash speed (~60-120 us).

---

## 5. NVLink vs. Spectrum-X: Complementary Roles

A common misconception is that NVLink and Spectrum-X are competing interconnect technologies. They serve fundamentally different purposes in the Vera Rubin architecture:

| Dimension | NVLink 6 | Spectrum-X Ethernet |
|-----------|----------|-------------------|
| **Primary function** | GPU-to-GPU within a rack | Rack-to-rack, GPU-to-storage, GPU-to-CPU racks |
| **Topology** | All-to-all within 72-GPU NVL72 | Fat-tree or Clos across racks |
| **Bandwidth per GPU** | 3,600 GB/s | 100-800 GB/s (depends on NIC count per node) |
| **Protocol** | Proprietary NVLink | Standard Ethernet (RoCEv2) |
| **Latency** | Sub-microsecond GPU-to-GPU | Low single-digit microseconds |
| **Scalability** | 72 GPUs (single rack) | Thousands of endpoints (multi-rack) |
| **Storage access** | No (does not connect to storage) | Yes (primary storage access fabric) |
| **Standards compliance** | NVIDIA proprietary | IEEE 802.3 Ethernet |

### 5.1 The Division of Labor

In the Vera Rubin POD, workload traffic naturally partitions across these two fabrics:

**NVLink 6 carries:**
- All-to-all expert routing in MoE models
- Gradient synchronization (allreduce) during training
- Tensor parallelism communication between GPUs within a node
- Pipeline parallelism activation transfers between stages within a rack

**Spectrum-X carries:**
- KV cache retrieval from CMX storage nodes (STX racks)
- Model weight loading from persistent storage
- Inter-rack GPU communication for data parallelism
- RL (reinforcement learning) orchestration traffic from Vera CPU racks
- Management, telemetry, and control plane traffic

### 5.2 Implications for Storage Engineers

The key insight is that **all storage traffic flows exclusively over Spectrum-X Ethernet**. NVLink, despite its vastly higher bandwidth, is invisible to the storage subsystem. This means:

1. **Storage bandwidth planning should use Spectrum-X numbers, not NVLink numbers.** An NVL72 rack's aggregate storage bandwidth is determined by the number of ConnectX-9 SuperNICs and their uplinks to the SPX fabric, not by the 260 TB/s NVLink domain.

2. **CMX storage nodes are sized to match Spectrum-X fabric bandwidth.** Each STX storage rack provides NVMe SSD capacity and throughput scaled to the 800GbE links connecting it to the SPX spine. Over-provisioning SSD throughput beyond what the network can deliver is wasteful.

3. **Fabric congestion affects storage more than compute.** Since NVLink handles the highest-bandwidth compute traffic within the rack, the Spectrum-X fabric is proportionally less loaded. However, during inference serving with heavy context retrieval, storage traffic can dominate the Ethernet fabric. Spectrum-X's adaptive routing and congestion control become critical to maintaining predictable storage access latency under load.

---

## 6. NetX Rack-Scale Networking: Connecting the Pod

### 6.1 The Five Rack Types

The Vera Rubin POD defines five distinct rack-scale systems, all built on the third-generation NVIDIA MGX modular architecture ([NVIDIA Vera Rubin POD](https://developer.nvidia.com/blog/nvidia-vera-rubin-pod-seven-chips-five-rack-scale-systems-one-ai-supercomputer/)):

| Rack Type | Function | Key Silicon | Quantity in POD |
|-----------|----------|-------------|-----------------|
| **DGX Vera Rubin NVL72** | GPU compute (training + inference) | 72 Rubin GPUs, 36 Vera CPUs, NVLink 6 Switch | 16 racks (8 SuperPODs) |
| **LPX** | Decode acceleration | 256 Groq 3 LPUs | Variable |
| **Vera CPU Rack** | RL orchestration, data processing | 256 Vera CPUs | Variable |
| **STX** | CMX context memory storage | BlueField-4 DPUs, NVMe SSDs | Variable |
| **SPX** | Ethernet networking spine | Spectrum-6 switches with CPO | Variable |

The full Vera Rubin POD comprises approximately 40 racks totaling 1,152 Rubin GPUs, nearly 20,000 NVIDIA dies, 1.2 quadrillion transistors, 60 exaflops of FP4 compute, and 10 PB/s of aggregate bandwidth.

### 6.2 SPX as the Fabric Spine

The SPX networking rack serves as the central interconnect hub for the entire pod. It contains Spectrum-6 switches with 102.4 Tb/s capacity and co-packaged optics, connecting:

- **NVL72 ↔ NVL72:** Inter-rack GPU communication for data parallelism, pipeline parallelism across racks, and multi-rack collective operations.
- **NVL72 ↔ STX:** GPU-to-CMX storage traffic for KV cache retrieval, context memory access, and checkpoint I/O.
- **NVL72 ↔ Vera CPU Rack:** Communication between GPU compute nodes and CPU-based RL training, data preprocessing, and orchestration.
- **NVL72 ↔ LPX:** Routing of inference requests between prefill (GPU) and decode (LPU) stages.
- **STX ↔ external storage:** Access to persistent storage beyond the pod (model repositories, training datasets).

### 6.3 Networking Bandwidth Budget

A useful exercise for storage engineers is to estimate the available storage bandwidth per NVL72 rack:

- Each NVL72 rack contains multiple ConnectX-9 SuperNICs for Spectrum-X uplinks (the exact count depends on configuration, but NVIDIA's reference design provisions multiple 800GbE uplinks per compute tray).
- With 36 compute trays per NVL72 (each containing 2 GPUs and 1 CPU) and assuming at minimum one 800GbE uplink per tray shared between scale-out compute traffic and storage traffic, the per-GPU Ethernet bandwidth is on the order of 50-100 GB/s.
- However, this bandwidth is shared between inter-rack compute traffic and storage access. During inference serving, where inter-rack compute traffic is lower than during training, more Ethernet bandwidth is available for CMX storage access.

The practical implication: an NVL72 rack running inference workloads can sustain aggregate storage read throughput in the range of hundreds of GB/s to the CMX tier, which must be matched by sufficient STX storage node provisioning behind the SPX fabric.

### 6.4 Seven Chips, One Coherent Fabric

The seven chips in the Vera Rubin platform — Rubin GPU, Vera CPU, NVLink 6 Switch, ConnectX-9 SuperNIC, BlueField-4 DPU, Spectrum-6 Switch, and Groq 3 LPU — are co-designed to work as a coherent system ([NVIDIA Vera Rubin Platform](https://nvidianews.nvidia.com/news/nvidia-vera-rubin-platform)). The networking stack is unified:

- **NVLink 6** provides the intra-rack GPU fabric.
- **ConnectX-9** provides the host-to-Ethernet bridge on compute and CPU nodes.
- **Spectrum-6** provides the Ethernet switching fabric.
- **BlueField-4** provides the storage-side Ethernet termination and NVMe management.

This vertical integration means that NVIDIA controls the entire data path from GPU to SSD, enabling end-to-end optimization that is impossible with commodity components. RDMA semantics are preserved across the full path. Telemetry flows from every component back to the centralized AI-driven network management. Congestion signals propagate from switch to NIC in hardware, without software involvement.

---

## 7. Key Takeaways

### For Storage System Architects

1. **The NVMe SSD is the latency bottleneck, not the network.** In the CMX data path, NVMe flash access (50-100 us for random reads) accounts for 80-90% of end-to-end latency. Network hops contribute only 10-15 microseconds combined. Invest in SSD-tier optimization: higher queue depths, intelligent prefetching, SLC caching for hot entries, and PCIe Gen6 drives like the Micron 9650.

2. **Storage bandwidth is bounded by Spectrum-X, not NVLink.** The 260 TB/s NVLink domain is irrelevant for storage planning. Size CMX storage racks to match the 800GbE Spectrum-X uplink capacity of the compute racks they serve. Approximately 4 PCIe Gen6 NVMe SSDs at peak sequential read rate saturate a single 800GbE link.

3. **RDMA zero-copy is not optional — it is the design assumption.** CMX's performance targets assume RDMA zero-copy data movement. Storage solutions that introduce additional memory copies (e.g., userspace buffering, kernel-space NVMe-oF target implementations) will not meet latency targets. BlueField-4's hardware KV I/O plane is purpose-built for this.

4. **Spectrum-X's adaptive routing and congestion control matter for storage QoS.** Unlike dedicated storage networks, the Spectrum-X fabric carries mixed traffic (compute + storage). Without adaptive routing and hardware congestion control, storage read tail latency would be unpredictable. Spectrum-X's per-packet load balancing and telemetry-based rate control are what make shared fabric storage access viable.

### For Capacity Planning

5. **STX rack count scales with inference context requirements.** Each STX rack provides a finite amount of NVMe capacity and throughput. As agentic AI workloads grow context windows (millions of tokens per session), the ratio of STX storage racks to NVL72 compute racks increases. NVIDIA's claim of 5x higher tokens-per-second with CMX versus traditional storage depends on having sufficient STX capacity provisioned.

6. **PCIe Gen6 is a transitional bottleneck.** At 64 GB/s per direction for a x16 link, PCIe Gen6 is narrower than the 100 GB/s 800GbE line rate in one direction. Future generations (PCIe Gen7, CXL 3.0) will widen this, but for 2026-2027 deployments, PCIe Gen6 is the point where system designers must carefully balance NIC port count, SSD count, and PCIe lane allocation.

7. **Co-packaged optics in Spectrum-6 change deployment economics.** The 5x power efficiency improvement and 10x resiliency improvement of CPO versus pluggable optics reduces the operational cost of large-scale Ethernet fabrics. For storage networks that must connect dozens of STX racks to hundreds of compute nodes, CPO makes dense, high-radix switch topologies economically viable.

### The Big Picture

The Vera Rubin platform's interconnect architecture is designed around a clear hierarchy: NVLink 6 provides massive bandwidth (3.6 TB/s per GPU) within the 72-GPU compute domain, while Spectrum-X Ethernet provides flexible, standards-based connectivity (800GbE per port) across the broader pod including storage. For storage engineers, Spectrum-X is the technology that determines CMX performance — its RoCE implementation, adaptive routing, and BlueField-4 integration define the achievable throughput and latency for every KV cache access. The NVLink domain is a compute concern; the Ethernet domain is where storage lives.

---

*Sources referenced in this report:*
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
