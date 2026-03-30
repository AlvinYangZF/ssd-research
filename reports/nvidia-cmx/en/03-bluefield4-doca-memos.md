# BlueField-4 and DOCA Memos: The Storage Processor Brain of NVIDIA CMX

## 1. Introduction

The NVIDIA BlueField-4 represents a fundamental shift in how storage infrastructure is architected for AI workloads. Where previous generations of data processing units (DPUs) focused on general-purpose infrastructure offload -- networking, security, virtualization -- BlueField-4 is purpose-built as a **storage processor** for inference-time KV cache management. It is the silicon brain of NVIDIA's Context Memory Extensions (CMX) platform, the component that transforms a rack of NVMe SSDs into a shared, pod-level context memory tier.

The evolution is significant. BlueField-2 introduced the concept of a programmable DPU that could offload SmartNIC, storage, and security functions from the host CPU. BlueField-3 scaled that vision to 400 Gb/s with more Arm cores and hardware accelerators, finding its footing in cloud-native infrastructure. BlueField-4 takes an entirely different trajectory: it integrates a full NVIDIA Grace CPU (64 Arm Neoverse V2 cores), pairs it with ConnectX-9 800 Gb/s networking silicon, and adds hardware acceleration engines specifically tuned for KV cache integrity, encryption, and NVMe management. The result is not merely a faster DPU -- it is a new category of storage processor designed to sit between GPU memory and persistent flash, managing the ephemeral context data that agentic AI workloads generate at scale.

NVIDIA announced BlueField-4 at CES 2026 in January and followed up with the BlueField-4 STX reference architecture at GTC 2026 in March. The chip is expected in early availability in the second half of 2026 as part of the Vera Rubin platform family. For storage engineers, BlueField-4 and its accompanying DOCA Memos SDK define a new operational model: storage that is purpose-built for KV cache, managed through key-value APIs rather than block or file abstractions, and orchestrated by inference frameworks like NVIDIA Dynamo rather than traditional storage controllers.

This report examines the hardware architecture, storage processor role, DOCA Memos SDK, and full software stack that together form the CMX storage plane.

Sources: [NVIDIA Newsroom: BlueField-4 Powers New Class of AI-Native Storage](https://nvidianews.nvidia.com/news/nvidia-bluefield-4-powers-new-class-of-ai-native-storage-infrastructure-for-the-next-frontier-of-ai), [NVIDIA Blog: BlueField-4 AI Factory](https://blogs.nvidia.com/blog/bluefield-4-ai-factory/), [NVIDIA Newsroom: BlueField-4 STX Architecture](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption)

---

## 2. BlueField-4 Hardware Architecture

### 2.1 Processor Complex: Grace CPU with 64 Arm Neoverse V2 Cores

BlueField-4's compute subsystem is built around an NVIDIA Grace CPU featuring 64 Arm Neoverse V2 cores. This is a dramatic leap from BlueField-3's 16 Arm Cortex-A78 cores -- a 4x increase in core count with a generational jump in microarchitecture. The Neoverse V2 cores support SVE2 (Scalable Vector Extension 2), deliver higher IPC (instructions per clock), and are fabricated on a process that pushes BlueField-4's total transistor count to 64 billion, roughly triple the 22 billion transistors in BlueField-3.

NVIDIA markets this as delivering "6x the compute power" of BlueField-3. For storage workloads, the additional compute headroom matters: it enables richer in-band analytics, real-time telemetry processing, smarter enclosure management, predictive maintenance, and dynamic workload placement -- all running on the DPU without consuming host CPU or GPU cycles.

| Specification | BlueField-3 | BlueField-4 |
|---|---|---|
| CPU cores | 16x Arm Cortex-A78 | 64x Arm Neoverse V2 |
| Transistors | 22 billion | 64 billion |
| Memory | 32 GB DDR5 | 128 GB LPDDR5 |
| Cache | 8 MB L2 | 114 MB shared L3 |
| On-board SSD | 128 GB | 512 GB |
| Network engine | ConnectX-7 | ConnectX-9 |
| Network bandwidth | 400 Gb/s | 800 Gb/s |
| Host interface | PCIe Gen 5 x16 | PCIe Gen 6 x16 |
| Compute multiplier | 1x (baseline) | 6x |

Sources: [HPCwire: NVIDIA Cranks BlueField-4 to 800 Gbps](https://www.hpcwire.com/2025/10/28/nvidia-cranks-its-bluefield-4-dpu-to-800-gbps/), [ServeTheHome: BlueField-4 64 Arm Cores](https://www.servethehome.com/nvidia-bluefield-4-with-64-arm-cores-and-800g-networking-announced-for-2026/), [Tom's Hardware: BlueField-3 and BlueField-4 DPUs](https://www.tomshardware.com/news/nvidia-unveils-bluefield-3-and-bluefield-4-dpus)

### 2.2 Memory Subsystem

BlueField-4 integrates 128 GB of LPDDR5 memory, a 4x increase over BlueField-3's 32 GB DDR5. The switch to LPDDR5 reflects both power efficiency goals and the bandwidth requirements of managing KV cache metadata at 800 Gb/s line rate. The 114 MB of shared L3 cache (versus 8 MB L2 on BlueField-3) provides a substantial working set for the NVMe controller firmware, DOCA runtime services, and encryption/integrity state machines.

The on-board 512 GB SSD (up from 128 GB) serves as the boot device and local scratch space for the DOCA operating environment, container images, and telemetry logs. This is distinct from the NVMe SSDs that BlueField-4 manages in a CMX enclosure -- those are external drives connected via the storage backplane.

### 2.3 ConnectX-9 Networking Engine: 800 Gb/s Ethernet and RDMA

The integrated ConnectX-9 SuperNIC engine delivers 800 Gb/s of aggregate network bandwidth, doubling BlueField-3's 400 Gb/s. It supports both InfiniBand and Ethernet at 800 Gb/s, with NVIDIA's Spectrum-X Ethernet fabric as the primary deployment target for CMX.

For storage engineers, the critical capability is **RDMA over Converged Ethernet (RoCE)**. CMX data nodes communicate with GPU compute nodes via RDMA, bypassing the host CPU entirely. The ConnectX-9 engine handles RDMA protocol termination in hardware, including NVMe-oF (NVMe over Fabrics) target termination. This means BlueField-4 can present NVMe namespaces over the network fabric without software-based protocol translation on the critical path.

The ConnectX-9 also supports advanced congestion control and adaptive routing features inherited from Spectrum-X, which are essential for maintaining consistent, low-jitter RDMA performance in multi-tenant pod environments where hundreds of GPU nodes may be contending for KV cache access simultaneously.

Sources: [NVIDIA ConnectX-9 SuperNIC Analysis](https://www.naddod.com/ai-insights/nvidia-connectx-9-supernic-analysis-1x800g-ethernet-breakthrough), [NVIDIA Ethernet SuperNICs](https://www.nvidia.com/en-us/networking/products/ethernet/supernic/)

### 2.4 PCIe Gen 6 Host Interface

BlueField-4 connects to its host (or, in headless CMX deployments, to the storage backplane) via a 16-lane PCIe Gen 6 interface. PCIe Gen 6 doubles the per-lane bandwidth from Gen 5's 32 GT/s to 64 GT/s, delivering approximately 128 GB/s of aggregate bidirectional bandwidth. This is relevant for CMX enclosures where BlueField-4 must sustain high-throughput NVMe command submission and completion across a large number of SSDs.

### 2.5 Hardware Acceleration Engines

BlueField-4 inherits and expands the hardware accelerator portfolio from previous generations:

- **Cryptographic acceleration**: Line-rate AES encryption/decryption at 800 Gb/s. This covers encryption at rest for KV cache stored on NVMe SSDs and encryption in flight for RDMA transfers. The crypto engine operates inline -- data passes through it without store-and-forward buffering -- so it adds negligible latency to the data path.

- **CRC / data integrity**: Hardware CRC computation at 800 Gb/s line rate. Used for T10-DIF style end-to-end data integrity protection on KV cache blocks. Every block written to NVMe is protected by a hardware-generated checksum; every block read back is verified before being sent over RDMA to the requesting GPU node.

- **Compression / decompression**: Hardware-assisted compression (documented in the DOCA/DPDK mlx5 compress driver) for reducing KV cache footprint on flash. This is particularly relevant for attention KV blocks, which exhibit compressibility ratios that can meaningfully extend effective flash capacity.

- **DMA engines**: High-performance DMA for moving data between host memory, DPU memory, NVMe backends, and the network engine without CPU involvement.

- **Regular expression (RegEx) acceleration**: Carried forward from BlueField-3, primarily used for deep packet inspection and security policy enforcement rather than storage operations.

These accelerators are critical because they allow BlueField-4 to perform encryption, integrity, and compression on KV cache data at full 800 Gb/s line rate **without consuming Grace CPU cycles**. The CPU remains available for control-plane logic, metadata management, telemetry, and running DOCA services.

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [DPDK mlx5 Compress Driver](https://doc.dpdk.org/guides/compressdevs/mlx5.html), [Chiplog: Analysis of BlueField-4](https://www.chiplog.io/p/analysis-of-nvidias-bluefield-4-dpu)

---

## 3. Storage Processor Role

### 3.1 NVMe SSD Management

In a CMX deployment, BlueField-4 acts as the NVMe controller for the SSDs in the enclosure. It does not merely pass through NVMe commands from a host -- it **owns** the NVMe namespaces, manages the command queues, handles wear leveling policies, and orchestrates data placement across the drive pool. The host CPU (if one exists in the enclosure) and GPU nodes on the network never interact with the SSDs directly; they interact with BlueField-4 through DOCA Memos APIs or NVMe-oF.

This is a fundamental architectural distinction from traditional DPU-based storage, where the DPU acts as a virtio-blk or NVMe SNAP emulation layer presenting remote storage as local drives. In CMX, the BlueField-4 is the **storage controller itself**. It runs NVMe driver logic natively, using SPDK (Storage Performance Development Kit) block device backends for NVMe namespace management.

BlueField-4 supports standard NVMe and NVMe-oF transports, including **NVMe Key-Value (KV) command set extensions**. The NVMe KV command set (defined in the NVMe specification) provides native key-value semantics at the protocol level -- put, get, delete, and list operations on variable-length keys and values -- which aligns naturally with the block-oriented nature of KV cache management. This is not an application-level abstraction layered on top of block storage; it is a protocol-level capability that the NVMe SSDs and BlueField-4's NVMe controller can jointly exploit.

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [NVIDIA BlueField-3 SNAP Documentation](https://docs.nvidia.com/networking/display/bluefield3snap440/introduction)

### 3.2 Data Integrity Offload

Data integrity for KV cache is not optional -- a single bit flip in a KV block can corrupt an entire inference session, producing nonsensical outputs or, worse, subtly wrong results. BlueField-4 addresses this with hardware-accelerated integrity checking at every stage of the data path:

- **Write path**: When a KV block arrives over RDMA from a GPU node, BlueField-4's CRC engine computes a checksum before writing the block to NVMe. The checksum is stored alongside the data, following a model analogous to T10-DIF (Data Integrity Field) end-to-end protection.

- **Read path**: When a KV block is read back from NVMe for delivery to a GPU node, the CRC engine verifies the stored checksum against the data. If the check fails, the block is flagged as corrupt, and the inference framework can recompute it rather than serving stale or damaged context.

- **In-flight protection**: RDMA transfers between BlueField-4 and GPU nodes are protected by the RoCE transport's own CRC mechanisms, providing network-layer integrity in addition to storage-layer integrity.

All of this happens at 800 Gb/s line rate in dedicated hardware. The Grace CPU is not involved in checksum computation or verification on the data path.

### 3.3 Encryption at Rest

BlueField-4's crypto acceleration engine provides AES-256 encryption for all KV cache data written to NVMe SSDs. In multi-tenant cloud environments, this ensures that KV cache belonging to one tenant cannot be read by another, even if physical drives are shared or decommissioned.

The encryption operates inline on the write path and decryption on the read path, both at 800 Gb/s. Key management is handled by the DOCA security services running on the Grace CPU, with hardware root-of-trust mechanisms inherited from previous BlueField generations ensuring that encryption keys are not exposed to the host or network.

For storage engineers accustomed to self-encrypting drives (SEDs), the BlueField-4 model is different: encryption is performed by the **storage processor**, not the drive itself. This gives the storage operator more flexibility -- different encryption keys per tenant, per session, or per KV namespace -- without depending on the SED capabilities of the underlying NVMe drives.

### 3.4 Compression for KV Cache

KV cache blocks generated by large language models during inference exhibit meaningful compressibility. Attention key and value tensors often contain redundant patterns, especially for long-context sessions where much of the context window is shared across turns.

BlueField-4's hardware compression engine can compress KV blocks before writing them to NVMe and decompress them on read-back. This effectively increases the usable capacity of the flash tier without requiring larger or more drives. The compression is transparent to the inference framework -- DOCA Memos handles it internally, presenting uncompressed KV blocks to the consumer.

The trade-off is latency: compression and decompression add processing time to the write and read paths. For KV cache workloads where the data is ephemeral and access patterns are sequential (prefill once, read during decode), the capacity benefit typically outweighs the latency cost, particularly when the alternative is a cache miss that forces full recomputation of the context.

### 3.5 Offloading from Host CPU and GPU

The core value proposition of BlueField-4 as a storage processor is offloading. In traditional storage architectures, the host CPU handles NVMe driver execution, checksum computation, encryption, compression, and network protocol processing. In GPU-centric AI clusters, consuming host CPU cycles for storage operations directly reduces the cycles available for inference orchestration.

BlueField-4 eliminates this overhead entirely. The GPU node's interaction with CMX storage is mediated by RDMA -- the GPU (or its host CPU) issues a DOCA Memos API call, which translates into an RDMA transfer to the BlueField-4 node. From that point, all NVMe command processing, integrity checking, encryption, and compression happen on BlueField-4's silicon. The GPU node's host CPU is not involved in any storage I/O processing.

NVIDIA quantifies this advantage as up to **5x tokens per second** and **4x better energy efficiency** compared with traditional CPU-based storage architectures, along with **2x page ingestion speed** for retrieval-augmented generation (RAG) workloads.

Sources: [Tom's Hardware: BlueField-4 STX for Agentic AI](https://www.tomshardware.com/tech-industry/nvidia-launches-bluefield-4-stx-storage-architecture-for-agentic-ai), [VentureBeat: BlueField-4 STX Context Memory](https://venturebeat.com/data/nvidia-bluefield-4-stx-adds-a-context-memory-layer-to-storage-to-close-the)

---

## 4. DOCA Memos SDK

### 4.1 Purpose and Positioning

DOCA Memos is the software interface between inference frameworks and the CMX storage tier. It is a BlueField-4- and CMX-optimized SDK within the broader DOCA (Data Center Infrastructure on a Chip Architecture) software framework. Its purpose is narrow and specific: **expose KV cache storage as a key-value service** that inference engines can consume without knowing or caring about the underlying NVMe drives, RDMA transports, or encryption machinery.

DOCA Memos turns Ethernet-attached flash into a pod-level cache tier. Applications remain stateless -- they put and get KV blocks through the API -- while CMX handles routing, placement, replication, integrity, and lifecycle management behind the scenes.

Sources: [NVIDIA CMX Context Memory Storage Platform](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/), [Blocks and Files: NVIDIA Context Memory Extension](https://blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/)

### 4.2 Key-Value API Surface

DOCA Memos exposes key-value semantics aligned with the NVMe KV command set and the operational model of inference KV cache management. The core operations are:

- **Put**: Store a KV cache block (or set of blocks) on the CMX tier, associated with a key that typically encodes the model ID, layer index, sequence hash, or session identifier. The put operation triggers the full write pipeline: integrity checksum generation, optional compression, encryption, and NVMe write to the target SSD(s).

- **Get**: Retrieve a KV cache block by key. The get operation reverses the write pipeline: NVMe read, integrity verification, decryption, optional decompression, and RDMA transfer to the requesting GPU node. If the block has been evicted or expired, the get returns a cache miss, signaling the inference framework to recompute the KV context.

- **Delete**: Explicitly remove a KV cache block. Used when an inference session ends, a context window is invalidated, or the framework determines that cached context is no longer reusable.

- **List**: Enumerate keys matching a pattern or prefix. Used by inference frameworks for cache discovery -- determining which KV blocks are already available on CMX before deciding whether to recompute or reuse context.

These operations map naturally to the NVMe KV command set extensions supported by BlueField-4's NVMe controller, enabling a clean translation from API call to protocol-level operation without an intermediate block-translation layer.

### 4.3 Cache Management: Eviction, TTL, and Priority

KV cache is ephemeral by nature. Unlike traditional storage data, KV blocks have a natural expiration -- they are useful only as long as the inference session or conversation they belong to remains active. DOCA Memos implements cache lifecycle management through several mechanisms:

- **TTL (Time-to-Live)**: KV blocks can be assigned a TTL at write time. When the TTL expires, the block becomes eligible for eviction. This aligns with NVIDIA Dynamo's TTL-based cache pinning model, where the inference router specifies how long to retain KV blocks for specific conversation turns or sessions.

- **LRU eviction**: Under memory pressure, DOCA Memos evicts blocks based on Least Recently Used (LRU) ordering. Blocks that have not been accessed recently are evicted first, freeing flash capacity for new writes.

- **Priority-based eviction**: Inference frameworks can assign priority levels to KV blocks. System-prompt KV cache (shared across many sessions) can be pinned at high priority, while per-turn context that is unlikely to be reused can be assigned low priority. Under eviction pressure, low-priority blocks are evicted before high-priority blocks, regardless of recency.

- **Tiered demotion**: Rather than deleting evicted blocks outright, DOCA Memos can demote blocks from GPU HBM to host/Grace CPU memory, and from CPU memory to CMX flash storage. This tiered approach, orchestrated jointly with NIXL, means that "eviction" from one tier is "ingestion" into the next, preserving the context for potential reuse at a lower performance tier.

These policies are configurable per namespace or per tenant, enabling multi-tenant deployments where different inference workloads have different cache retention requirements.

Sources: [NVIDIA Developer Blog: KV Cache Reuse in TensorRT-LLM](https://developer.nvidia.com/blog/introducing-new-kv-cache-reuse-optimizations-in-nvidia-tensorrt-llm/), [NVIDIA Developer Blog: Reducing KV Cache Bottlenecks with Dynamo](https://developer.nvidia.com/blog/how-to-reduce-kv-cache-bottlenecks-with-nvidia-dynamo/), [NAND Research: NVIDIA STX and CMX](https://nand-research.com/nvidia-stx-cmx-infrastructure-for-agentic-ai-context-storage/)

### 4.4 Multi-Tenancy

In cloud and multi-tenant deployments, CMX must isolate KV cache between tenants at every level: network, storage, and API. DOCA Memos supports multi-tenancy through:

- **Namespace isolation**: Each tenant (or inference service) operates within a dedicated KV namespace. Namespaces are enforced at the BlueField-4 level -- a tenant's API credentials grant access only to their namespace, and cross-namespace access is prohibited by the DOCA runtime.

- **Per-tenant encryption keys**: BlueField-4's crypto engine supports per-namespace encryption keys, ensuring that even if physical NVMe capacity is shared across tenants, data at rest is cryptographically isolated.

- **Network-level isolation**: BlueField DPUs authenticate into each tenant's VPN or network segment, presenting isolated virtual NICs. The DPU ensures that KV cache traffic from one tenant cannot be observed or intercepted by another tenant on the same physical infrastructure.

- **Quality-of-service (QoS)**: DOCA Memos can enforce per-tenant bandwidth and IOPS limits, preventing a noisy-neighbor inference workload from starving other tenants of CMX throughput.

This multi-tenancy model extends to the Spectrum-X Ethernet fabric, which provides advanced congestion control and adaptive routing to maintain consistent, predictable RDMA performance even when many tenants share the same pod-level CMX infrastructure.

### 4.5 Inference Framework Integration

DOCA Memos is designed to integrate with the NVIDIA inference software stack, specifically:

- **NVIDIA Dynamo**: The distributed inference framework that orchestrates KV cache placement and movement across memory tiers. Dynamo's KV block manager uses DOCA Memos as the API for CMX storage operations. When Dynamo decides to offload KV blocks from GPU HBM to flash, it issues put operations through DOCA Memos. When it needs to prestage KV blocks for an upcoming decode phase, it issues get operations.

- **TensorRT-LLM**: NVIDIA's optimized inference engine for large language models. TensorRT-LLM's KV cache reuse optimizations -- including prefix caching and cross-request cache sharing -- benefit directly from DOCA Memos. When multiple inference requests share a common system prompt, the KV cache for that prompt can be stored once on CMX and retrieved by any GPU node, eliminating redundant recomputation.

- **vLLM**: The open-source inference engine's PagedAttention mechanism manages KV cache in GPU memory with near-zero waste. DOCA Memos extends this model to flash storage -- when GPU memory is full, vLLM can page KV blocks out to CMX through DOCA Memos and page them back in when needed.

- **NVIDIA NIXL (Inference Transfer Library)**: NIXL is the transport layer that DOCA Memos relies on for moving KV blocks between memory tiers. It provides a unified API regardless of whether the transfer traverses NVLink, InfiniBand, RoCE, or Ethernet, and supports asynchronous operation so that GPU computation is not stalled during KV transfers.

The integration model is layered: inference frameworks call Dynamo's KV manager, which calls DOCA Memos APIs, which translate into NIXL transfers and NVMe operations on BlueField-4. Each layer abstracts the complexity below it.

Sources: [NVIDIA Developer Blog: Dynamo Low-Latency Inference](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/), [NVIDIA Developer Blog: KV Cache Bottlenecks with Dynamo](https://developer.nvidia.com/blog/how-to-reduce-kv-cache-bottlenecks-with-nvidia-dynamo/), [Blocks and Files: NVIDIA KV Cache Offload to NVMe](https://blocksandfiles.com/2026/01/06/nvidia-standardizes-gpu-cluster-kv-cache-offload-to-nvme-ssds/)

---

## 5. Software Stack Architecture

### 5.1 DOCA Runtime on BlueField-4

BlueField-4 runs a full Linux operating environment on its 64 Arm Neoverse V2 cores, with the DOCA runtime as the primary software platform. The DOCA runtime consists of:

- **Operating system**: A standard Linux distribution (typically Ubuntu-based) optimized for the Arm architecture, running directly on the Grace CPU cores within BlueField-4.

- **DOCA SDK**: Industry-standard drivers and libraries including DPDK (Data Plane Development Kit) for high-performance packet processing, SPDK (Storage Performance Development Kit) for NVMe device management, and P4 programmability for custom packet processing pipelines.

- **DOCA microservices**: Containerized services that run on BlueField-4 and provide networking, storage, security, and telemetry functions. In CMX deployments, the key microservice is the DOCA Memos service, which implements the KV cache management logic and exposes the key-value API to the network.

- **Container orchestration**: The DOCA Platform (available as an open-source project on GitHub) manages provisioning and service orchestration for BlueField DPUs across a data center. It supports deploying, updating, and monitoring DOCA microservices at scale -- critical when a CMX deployment may include hundreds of BlueField-4 processors across multiple enclosures.

The DOCA runtime on BlueField-4 is fully self-contained. It boots from the on-board 512 GB SSD, runs its own network stack, and manages the NVMe SSDs in the enclosure independently of any host operating system. In headless CMX enclosures (where there is no host CPU), BlueField-4 **is** the enclosure controller.

Sources: [NVIDIA DOCA Software Framework](https://developer.nvidia.com/networking/doca), [NVIDIA DOCA Documentation](https://docs.nvidia.com/doca/sdk/index.html), [GitHub: NVIDIA DOCA Platform](https://github.com/NVIDIA/doca-platform)

### 5.2 RDMA Communication with GPU Nodes

The data path between GPU compute nodes and CMX storage nodes is built on RDMA. The flow for a KV cache read operation looks like this:

1. The inference framework (e.g., Dynamo running on a GPU node) determines that it needs a KV block that is stored on CMX.
2. The framework calls the DOCA Memos client library, passing the key for the desired KV block.
3. The DOCA Memos client invokes NIXL to initiate an RDMA read from the CMX node.
4. On the CMX node, BlueField-4's ConnectX-9 engine receives the RDMA request and routes it to the DOCA Memos service.
5. The DOCA Memos service looks up the key in its metadata index, identifies the NVMe location, and issues an NVMe read through SPDK.
6. The data flows from the NVMe SSD through BlueField-4's integrity verification engine (CRC check), decryption engine (AES-256), and optional decompression engine.
7. The processed data is placed into an RDMA send buffer and transmitted back to the GPU node over the Spectrum-X Ethernet fabric.
8. On the GPU node, NIXL delivers the KV block directly into GPU HBM (or host memory) via GPUDirect RDMA.

The entire operation bypasses the CMX node's Grace CPU on the data path -- steps 5 through 7 are handled by hardware accelerators and DMA engines. The Grace CPU is involved only in the metadata lookup (step 5) and control-plane orchestration.

For write operations, the flow is reversed: the GPU node pushes KV blocks to CMX via RDMA, BlueField-4 applies compression, encryption, and CRC generation in hardware, and writes the result to NVMe.

### 5.3 NVMe Driver Interaction

BlueField-4 manages NVMe SSDs through SPDK, bypassing the kernel's NVMe driver entirely. SPDK provides user-space NVMe drivers that poll completion queues directly, eliminating interrupt overhead and context-switch latency. This is essential for the latency-sensitive KV cache workload, where a single get operation must complete in microseconds to avoid stalling the inference pipeline.

The NVMe interaction model in CMX has several distinctive characteristics:

- **NVMe KV command set**: Where supported by the underlying SSDs, BlueField-4 can use the NVMe KV command set (KV Retrieve, KV Store, KV Delete, KV List) natively, mapping DOCA Memos API calls directly to NVMe KV commands. This eliminates the key-to-LBA translation layer that would be required with block-based NVMe.

- **NVMe-oF target**: BlueField-4 can also expose NVMe namespaces over NVMe-oF (Fabrics), enabling remote GPU nodes to access SSDs via standard NVMe-oF initiators. This provides a fallback path for frameworks that prefer block-level access over key-value access.

- **SNAP emulation**: For backward compatibility with hosts expecting local NVMe devices, BlueField-4 supports SNAP (Software-defined Network Accelerated Processing), which emulates local NVMe controllers on the PCIe bus. In CMX, this is less relevant since the primary access model is network-based, but it is available for hybrid deployments.

- **Drive health management**: BlueField-4 monitors SSD health telemetry (SMART attributes, wear indicators, error rates) and can proactively migrate data away from failing drives. The 64-core Grace CPU provides ample headroom for running predictive maintenance algorithms without affecting data-path performance.

### 5.4 The G3.5 Storage Tier

CMX establishes what NVIDIA calls the "G3.5" storage tier in the AI data center memory hierarchy:

| Tier | Medium | Scope | Latency | Capacity |
|---|---|---|---|---|
| G1 | GPU HBM | Per-GPU | ~nanoseconds | 10s of GB |
| G2 | Host/Grace CPU DRAM | Per-node | ~100s of ns | 100s of GB |
| G3 | Local NVMe | Per-node | ~microseconds | TBs |
| **G3.5** | **CMX (pod-level flash)** | **Pod-wide** | **~10s of us** | **Petabytes** |
| G4 | Shared enterprise storage | Cluster-wide | ~milliseconds | Exabytes |

The G3.5 tier is uniquely positioned: it offers the capacity and persistence of shared storage with the low latency and high bandwidth of local flash, accessible across all nodes in a pod via RDMA. For KV cache -- which is ephemeral, high-bandwidth, and benefits enormously from sharing across inference sessions -- G3.5 fills a gap that neither local SSDs nor enterprise storage arrays can address efficiently.

DOCA Memos and NIXL jointly manage the movement of KV blocks across these tiers. When GPU HBM is full, KV blocks are demoted to G2 (host DRAM), then to G3.5 (CMX flash). When a KV block is needed for an upcoming decode phase, it is prestaged from G3.5 back into G2 or G1 ahead of time, hiding the flash access latency behind prefetching.

Sources: [NVIDIA Developer Blog: BlueField-4 ICMS](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [NAND Research: Improving Inference with NVIDIA ICMS](https://nand-research.com/research-note-improving-inference-nvidias-inference-context-memory-storage-platform/)

### 5.5 Monitoring and Telemetry

BlueField-4's 64-core Grace CPU provides substantial headroom for telemetry and monitoring services that run alongside the data-path workload. The DOCA runtime supports:

- **Drive telemetry**: Real-time collection of NVMe SMART data, wear leveling statistics, error rates, and thermal readings from all managed SSDs. This data feeds into predictive maintenance models that can preemptively redistribute data before a drive fails.

- **KV cache analytics**: Metrics on cache hit rate, eviction rate, put/get latency distributions, per-tenant utilization, and compression ratios. These metrics are essential for tuning eviction policies, TTL values, and capacity planning.

- **Network telemetry**: Spectrum-X fabric metrics including RDMA throughput, latency histograms, congestion events, and adaptive routing decisions. BlueField-4 can correlate storage and network telemetry to identify bottlenecks -- for example, detecting when KV cache miss latency is dominated by network congestion rather than NVMe access time.

- **Enclosure management**: Power consumption, thermal status, fan speeds, and hardware health for the CMX enclosure. BlueField-4 acts as the baseboard management controller for headless enclosures, providing out-of-band management interfaces.

The telemetry data can be exported via standard protocols (Prometheus, gRPC, SNMP) to data center monitoring platforms, enabling CMX storage infrastructure to be managed alongside traditional storage arrays and network equipment in unified operations dashboards.

Sources: [VAST Data: Advancing State of the Art with BlueField-4](https://www.vastdata.com/blog/advancing-the-state-of-the-art-with-nvidia-blue-field-4), [SiliconANGLE: Convergence of Context with BlueField-4 STX](https://siliconangle.com/2026/03/20/convergence-context-nvidias-bluefield-4-stx-marries-network-storage-admin/)

---

## 6. Key Takeaways

**BlueField-4 is not a NIC with a sidecar CPU -- it is a full storage processor.** The 64-core Grace CPU, 128 GB LPDDR5, and 800 Gb/s ConnectX-9 engine make it capable of independently managing NVMe enclosures, running containerized storage services, and handling all data-path operations (encryption, integrity, compression) in hardware at line rate. Storage engineers should think of it as an embedded storage controller, not a network adapter.

**DOCA Memos abstracts NVMe as a key-value service.** The traditional storage stack -- block devices, file systems, volume managers -- is replaced by a purpose-built KV API optimized for inference context. This is a paradigm shift for storage operations: provisioning, monitoring, and capacity planning are done in terms of KV namespaces, eviction policies, and TTLs rather than LUNs, RAID groups, and file system extents.

**The G3.5 tier changes capacity planning.** CMX creates a new tier in the storage hierarchy that did not exist before -- pod-level flash with RDMA access. Storage engineers must now plan for this tier alongside local SSDs (G3) and shared storage (G4), considering factors like KV block size, session duration, cache hit rates, and inter-pod traffic patterns.

**Hardware offload is the performance enabler.** The 5x throughput improvement over CPU-based storage is not from faster SSDs -- it is from eliminating CPU overhead on the data path. Every storage operation (checksum, encrypt, compress, NVMe command) that BlueField-4 handles in hardware is a cycle that the GPU node's host CPU does not spend, directly translating to higher inference throughput.

**Multi-tenancy is built in, not bolted on.** Per-tenant encryption keys, namespace isolation, network segmentation, and QoS enforcement are native to the BlueField-4 and DOCA Memos architecture. This is a requirement for cloud providers deploying shared CMX infrastructure across multiple inference customers.

**The ecosystem is the moat.** At GTC 2026, NVIDIA announced BlueField-4 STX adoption from ten major storage manufacturers. DOCA Memos integrates natively with Dynamo, TensorRT-LLM, vLLM, and NIXL. For storage engineers evaluating CMX, the question is not just whether the hardware meets requirements -- it is whether the inference software stack they deploy will assume CMX is present and optimize accordingly.

Sources: [NVIDIA Newsroom: BlueField-4 STX Broad Industry Adoption](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption), [SDxCentral: BlueField-4 First Look](https://www.sdxcentral.com/analysis/nvidias-bluefield-4-a-first-look-at-the-dpu-built-to-run-ai-factories/)
