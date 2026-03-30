# NVIDIA CMX Context Memory Platform: A Deep Technical Architecture Analysis

## 1. Introduction --- The KV Cache Problem: Why AI Inference Needs a New Memory Tier

The shift from single-turn chatbot interactions to long-context reasoning and multi-step agentic workflows has exposed a fundamental infrastructure bottleneck in AI inference: the key-value (KV) cache. Every token a large language model (LLM) processes generates key and value tensors that must be retained for subsequent attention calculations. As context windows expand from 8K to 128K and now to 1M+ tokens, and as agentic AI systems chain dozens of reasoning steps within a single session, the KV cache has grown from a minor bookkeeping overhead into the dominant consumer of GPU memory --- frequently exceeding the memory footprint of the model weights themselves.

This is not a software optimization problem that can be solved with better algorithms alone. It is a storage architecture problem. GPU high-bandwidth memory (HBM) is expensive, power-hungry, and capacity-constrained. Existing offload strategies --- spilling to host DRAM or dumping to general-purpose shared storage --- introduce latency cliffs that stall the GPU decode pipeline and destroy token throughput. The industry needs a purpose-built memory tier that sits between GPU HBM and enterprise storage: one that understands the ephemeral, block-structured nature of KV cache data and can serve it at the bandwidth and latency the decode pipeline demands.

NVIDIA's answer is the **CMX Context Memory Storage Platform**, announced at GTC 2026 as the first rack-scale implementation of the broader STX (Storage) architecture. CMX introduces a new "G3.5" tier --- Ethernet-attached flash storage powered by BlueField-4 DPUs --- designed specifically for pod-level KV cache storage, sharing, and retrieval. This report provides a storage-engineer-oriented deep dive into CMX's architecture, data flow, performance claims, and competitive positioning.

---

## 2. The KV Cache Challenge

### 2.1 What Is the KV Cache?

During transformer inference, the attention mechanism requires access to the key (K) and value (V) projections of every previously processed token. Rather than recomputing these projections at every generation step, inference engines cache them in GPU memory. Each transformer layer produces its own set of K and V tensors for every token in the sequence, and these must all be retained for the duration of that sequence's active inference. The KV cache grows linearly with both context length and model depth (number of layers and attention heads), and it must remain accessible at nanosecond-to-microsecond latency for the decode phase to proceed without stalling.

The formula for KV cache size per token is approximately:

```
KV cache per token = 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
```

For a model like Llama 3.1-70B with 80 layers, 8 KV heads per layer, and 128-dimensional heads in FP16 (2 bytes), each token consumes approximately 2 x 80 x 8 x 128 x 2 = 327,680 bytes (~320 KB) of KV cache. At 128K tokens, that is approximately 40 GB per request --- for a single user.

The memory cost is substantial and model-dependent:

| Model | Context Length | KV Cache per Request | Notes |
|---|---|---|---|
| Llama 3.1-70B | 128K tokens | ~40 GB HBM | Single user, FP16 |
| 70B-class model | 8K tokens | ~20 GB per request | ~640 GB for batch of 32 |
| 70B-class model | 1M tokens | ~15 GB per user (quantized) | With aggressive KV quantization |

Sources: [Pure Storage](https://blog.purestorage.com/perspectives/cut-llm-inference-costs-with-kv-caching/), [Introl](https://introl.com/blog/long-context-llm-infrastructure-million-token-windows-guide), [TensorWave](https://tensorwave.com/blog/estimating-llm-inference-memory-requirements)

At production concurrency levels (hundreds to thousands of simultaneous sessions), KV cache demand can reach **tens of terabytes** across a GPU pod. A single NVIDIA Blackwell B200 GPU has 192 GB of HBM3e. Even a rack of 72 GPUs (an NVL72 configuration) provides only ~14 TB of aggregate HBM. When multi-turn agents maintain persistent context across many reasoning steps, that capacity is consumed rapidly.

### 2.2 Current Approaches and Their Limitations

**GPU HBM Only:** The highest-performance option, but capacity is strictly bounded. Operators must choose between supporting fewer concurrent users or truncating context windows --- both of which degrade the quality and economics of inference services.

**CPU DRAM Offload:** Host system memory (DDR5/LPDDR5) offers 1--2 TB per node at lower cost per gigabyte than HBM. However, the PCIe bus between GPU and host memory introduces microsecond-scale latency and limited bandwidth compared to HBM. NVIDIA's own benchmarks show that CPU-GPU memory sharing via PCIe can accelerate KV cache offload, but it remains a node-local solution with no sharing across the pod. [NVIDIA Technical Blog](https://developer.nvidia.com/blog/accelerate-large-scale-llm-inference-and-kv-cache-offload-with-cpu-gpu-memory-sharing/)

**Local NVMe SSDs (G3 Tier):** Local SSDs offer terabytes of capacity at low cost, but they are node-local. KV cache stored on one node's SSDs cannot be efficiently accessed by GPUs on another node. This creates "stranded capacity" --- one node may have ample warm KV cache on its SSDs while another node handling a related request must recompute the same context from scratch. Research from Vrije Universiteit Amsterdam characterizing I/O patterns of KV cache offloading to NVMe SSDs found that while sequential write throughput is adequate, the read patterns during decode can exhibit high tail latency when the SSD controller is simultaneously handling writes from other sessions. [CHEOPS 2025 Workshop](https://atlarge-research.com/pdfs/2025-cheops-llm.pdf)

**Shared Enterprise Storage (G4 Tier):** Traditional NAS/object storage (NetApp, VAST, WEKA, etc.) provides petabyte-scale shared capacity with data protection, but millisecond-level access latency makes it unsuitable for active inference context. It is designed for durable data --- checkpoints, datasets, logs --- not for the ephemeral, high-churn KV blocks that inference demands.

**CXL-Attached Memory:** Compute Express Link (CXL) memory pooling is an emerging alternative that offers cache-coherent access to shared DRAM pools. Research from XConn Technologies and MemVerge demonstrates greater-than-5x performance improvements over SSD-based KV caching for offload scenarios. CXL 4.0 targets 1.5 TB/s aggregate bandwidth across pooled memory. However, CXL infrastructure remains pre-production for most deployments, requires specialized switches, and does not yet operate at pod scale over Ethernet. [XConn/MemVerge](https://www.prweb.com/releases/xconn-technologies-and-memverge-to-deliver-breakthrough-scalable-cxl-memory-solution-to-offload-kv-cache-and-prefilldecode-disaggregation-in-ai-inference-workloads-302616566.html), [Introl CXL Guide](https://introl.com/blog/cxl-4-0-infrastructure-planning-guide-memory-pooling-2025)

The gap is clear: there is no production-ready, pod-scale, shared memory tier that is optimized for the specific access patterns of KV cache. This is the gap CMX is designed to fill.

---

## 3. CMX Architecture

### 3.1 NVIDIA's Memory Hierarchy for AI Factories

NVIDIA has formalized a five-tier memory and storage hierarchy for AI inference infrastructure. Understanding these tiers is essential for positioning CMX:

| Tier | Medium | Scope | Latency Class | Role |
|---|---|---|---|---|
| **G1** | GPU HBM | Per-GPU | Nanoseconds | Active KV cache in token generation |
| **G2** | System DRAM | Per-node | Microseconds | Staging/buffering KV overflow from HBM |
| **G3** | Local NVMe SSDs | Per-node | ~100 us | Warm KV cache, short-timescale reuse |
| **G3.5** | **CMX (Ethernet-attached flash)** | **Pod-level** | **~100--500 us** | **Shared KV cache, cross-node context reuse** |
| **G4** | Shared enterprise storage | Cluster/DC | Milliseconds | Cold artifacts, checkpoints, durable data |

Source: [NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [Blocks and Files](https://www.blocksandfiles.com/2026/01/12/nvidias-basic-context-memory-extension-infrastructure/4090541)

CMX fills the G3.5 position: it provides **shared, Ethernet-attached flash capacity at the pod level**, purpose-built for the ephemeral, block-structured access patterns of KV cache. Unlike G3 (node-local SSDs), CMX makes KV blocks accessible to any GPU in the pod. Unlike G4 (shared storage), CMX is optimized for sub-millisecond access with hardware-accelerated data integrity rather than enterprise data protection semantics.

### 3.2 CMX Node Composition

Each CMX node is built around three core components:

**BlueField-4 Storage Processor.** The BlueField-4 is the computational heart of CMX. It integrates:

- A **64-core NVIDIA Grace CPU** (Arm Neoverse V2), a 4x core count increase over BlueField-3's 16-core Arm Cortex-A78, delivering approximately 6x the compute power of its predecessor
- **ConnectX-9 SuperNIC** providing **800 Gbps** Ethernet throughput (doubled from BlueField-3's 400 Gbps)
- **128 GB LPDDR5** memory and **114 MB shared L3 cache** for metadata, indexing, and DMA staging
- **512 GB on-board SSD** for local firmware and management
- Hardware acceleration engines for **line-rate encryption and CRC data protection at 800 Gbps**, ensuring data integrity without CPU overhead

Source: [Tom's Hardware](https://www.tomshardware.com/tech-industry/nvidia-launches-bluefield-4-stx-storage-architecture-for-agentic-ai), [ServeTheHome](https://www.servethehome.com/nvidia-bluefield-4-with-64-arm-cores-and-800g-networking-announced-for-2026/)

**NVMe SSDs.** BlueField-4 directly manages attached NVMe drives, handling data placement, wear leveling coordination, and I/O scheduling. The SSDs serve as the persistent (but ephemeral-lifecycle) backing store for KV cache blocks. NVIDIA has not specified a fixed drive count per CMX node, as this depends on the system vendor's chassis design. Reference platforms from Supermicro, QCT, and AIC are expected to support high-density NVMe configurations. For a deeper analysis of SSD vendor readiness and drive-level specifications relevant to CMX, see [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md).

**Spectrum-X Ethernet Fabric.** CMX nodes connect to the GPU compute pod via NVIDIA Spectrum-X switches running **RDMA over Converged Ethernet (RoCE)**. This provides the low-latency, high-bandwidth data path required for KV cache transfer without the cost and complexity of InfiniBand. Spectrum-X's adaptive routing and congestion control are tuned for the bursty, small-block I/O patterns characteristic of KV cache reads. The choice of Ethernet over InfiniBand is strategically significant: it allows CMX to be deployed in data centers that have standardized on Ethernet switching infrastructure, which includes the vast majority of cloud and enterprise environments. Spectrum-X adds RDMA capabilities and lossless flow control on top of standard Ethernet, providing InfiniBand-class performance within the pod without requiring a fabric technology change.

### 3.3 Software Stack

The CMX software stack consists of three interlocking layers:

**DOCA Memos.** This is a BlueField-4- and CMX-optimized SDK that exposes **simple key-value APIs** for KV cache management, sharing, and placement. DOCA Memos treats context cache as a first-class resource type, leveraging the unique properties of KV blocks and inference patterns. It handles KV cache routing and reuse at scale, allowing inference applications to remain stateless while CMX manages the underlying storage. DOCA Memos provides:

- KV block put/get/delete operations
- Hash-based block addressing for deduplication and lookup
- Secure, isolated access with hardware-accelerated integrity checks and encryption
- Metadata management for KV block lifetime, ownership, and sharing policies

Source: [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-bluefield-4-powers-new-class-of-ai-native-storage-infrastructure-for-the-next-frontier-of-ai)

**NIXL (NVIDIA Inference Transfer Library).** An open-source, high-performance library for accelerating point-to-point data movement between heterogeneous memory and storage tiers. NIXL provides the transport layer that moves KV blocks between GPU HBM (G1), system DRAM (G2), local SSDs (G3), CMX (G3.5), and shared storage (G4). It abstracts the underlying transport --- whether PCIe, NVLink, or RoCE --- into a unified transfer API. [NVIDIA NIXL](https://blockchain.news/news/nvidia-nixl-open-source-inference-transfer-library)

**NVIDIA Dynamo.** The distributed inference serving framework that orchestrates the entire pipeline. Dynamo's relevant components for CMX include:

- **KV-Aware Router:** Evaluates incoming requests against the existing KV cache state across the pod. It calculates the cost of routing a request to each worker based on KV cache overlap (cache hit ratio), prefill cost (new tokens to compute), and decode cost (active blocks). This minimizes redundant KV recomputation across the fleet. [NVIDIA Dynamo Docs](https://docs.nvidia.com/dynamo/latest/user-guides/kv-cache-aware-routing)
- **KV Block Manager:** A cost-aware caching engine that decides when and where to transfer KV blocks across the memory hierarchy. It handles eviction from G1 to G2/G3.5, promotion from G3.5 back to G1, and garbage collection of expired context.
- **SLO Planner:** Monitors capacity utilization and prefill activity, dynamically adjusting GPU resource allocation to meet latency service-level objectives.
- **Disaggregated Prefill/Decode:** Separates the compute-intensive prefill phase (processing the input prompt) from the memory-intensive decode phase (generating output tokens), allowing each to be scheduled on optimally configured hardware.

Source: [NVIDIA Dynamo Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)

### 3.4 Pod-Level Context Sharing

The defining architectural distinction of CMX versus node-local SSD storage (G3) is **pod-wide shared access**. Any GPU node in the pod can read or write KV blocks to any CMX node over the Spectrum-X RoCE fabric. This enables several critical capabilities:

- **Cross-node KV reuse:** If User A's session generated a KV cache for a 500K-token document, and User B submits a query against the same document, the KV cache can be served from CMX to User B's GPU without recomputation.
- **Agent state sharing:** In agentic AI workloads, multiple specialized agents may need access to the same reasoning context. CMX provides a shared namespace where agents can read each other's KV state without point-to-point transfers between GPU nodes.
- **Load-balanced eviction:** When one GPU node's HBM is under pressure, its KV blocks can be evicted to CMX and later retrieved by a different node that has been assigned the continuation of that session.

CMX offers **petabytes of shared capacity per GPU pod**, a dramatic expansion over the tens of terabytes available in aggregate HBM. [NVIDIA CMX Product Page](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/)

---

## 4. Data Flow

### 4.1 Request Arrival and KV Cache Lookup

When an inference request arrives at the Dynamo serving layer, the following sequence executes:

1. **Prompt hashing and KV index lookup.** The Dynamo KV-Aware Router computes a hash of the incoming prompt's token sequence (or prefix segments thereof). This hash is checked against the distributed KV cache index, which tracks which KV blocks exist, where they reside (G1, G2, G3, G3.5), and their freshness.

2. **Cache hit path (full or partial).** If matching KV blocks are found:
   - Blocks in G1 (HBM on the same GPU): Used directly. Zero transfer overhead.
   - Blocks in G2 (host DRAM on the same node): Transferred to G1 via PCIe/NVLink. Microsecond latency.
   - Blocks in G3.5 (CMX): NIXL initiates an RDMA read over the Spectrum-X fabric from the CMX node's NVMe-backed buffer into the target GPU's HBM. The BlueField-4 on the CMX node handles the NVMe read, CRC verification, and RDMA transmission.
   - The prefill phase is shortened or eliminated entirely for the cached prefix. NVIDIA reports a **20x improvement in time-to-first-token (TTFT)** when fetching KV cache from storage versus recomputing it. [VAST Data](https://www.vastdata.com/blog/how-nvidia-dynamo-vast-unlock-context-reuse-at-scale)

3. **Cache miss path.** If no matching KV blocks exist:
   - The request is routed to a prefill worker, which computes the KV cache from the input tokens.
   - The newly computed KV blocks are written to G1 (active use) and asynchronously written to G3.5 (CMX) for future reuse.

### 4.2 Write Path: From GPU HBM to Flash

KV cache writes to CMX follow this sequence:

1. **Eviction trigger.** The KV Block Manager on the GPU node determines that HBM pressure requires evicting cold KV blocks, or that newly generated KV blocks should be persisted to CMX for sharing.
2. **NIXL transfer.** NIXL initiates an RDMA write from GPU HBM (or host DRAM staging buffer) to the target CMX node. The write is directed to a specific BlueField-4 endpoint.
3. **BlueField-4 processing.** The BlueField-4 on the CMX node receives the RDMA payload, computes CRC integrity checks at line rate (800 Gbps), optionally encrypts the data, and issues NVMe write commands to the attached SSDs.
4. **Index update.** The KV block's metadata (hash, location, timestamp, ownership) is updated in the distributed index so that future lookups can locate it.
5. **Acknowledgment.** The CMX node acknowledges the write to the GPU node, which can then release the HBM capacity.

From a storage perspective, the write path is designed to be **asynchronous and non-blocking** relative to the decode pipeline. The GPU does not wait for the NVMe write to complete before proceeding with token generation. The BlueField-4's LPDDR5 buffer absorbs the write burst, and the NVMe flush happens in the background. This is critical because the decode phase is latency-sensitive --- any synchronous I/O in the write path would directly reduce token generation throughput.

The NVMe write pattern is predominantly **large sequential** (KV blocks are typically tens of megabytes), which aligns well with SSD controller optimization for sequential workloads. However, the high churn rate of KV blocks (written once, read a few times, then discarded within minutes to hours) creates elevated write amplification on flash media. SSD vendors targeting CMX workloads will need to optimize garbage collection and over-provisioning strategies for this ephemeral-write pattern.

### 4.3 Prefetch and Predictive Staging

A key optimization in the CMX architecture is **predictive prefetching**. Because the prompt and context pass through the Dynamo aggregator before reaching the LLM, the system can predict which KV blocks will be needed for upcoming decode steps. The aggregator sends hints to the CMX storage engine to prefetch KV entries from NVMe into the BlueField-4's LPDDR5 staging buffer, so that when the GPU issues the RDMA read, the data is already in fast memory rather than waiting on an NVMe read.

This prefetch mechanism is critical for maintaining decode throughput. The decode phase generates tokens one at a time and is highly latency-sensitive --- a single stall while waiting for a KV block to be read from SSD can propagate into visible generation pauses. By staging KV blocks into BlueField-4's 128 GB LPDDR5 buffer ahead of demand, CMX converts what would be a synchronous NVMe read (tens of microseconds) into a fast memory read (single-digit microseconds).

### 4.4 Multi-Turn Persistence

For multi-turn conversations and agentic workflows, CMX provides **session-level KV persistence**:

- After each turn, the accumulated KV cache is retained in CMX rather than discarded.
- When the user (or agent) returns with the next turn, the Dynamo router locates the session's KV blocks in CMX and prefetches them to the assigned GPU.
- This eliminates the need to reprocess the entire conversation history at each turn --- a cost that grows linearly with conversation length and can dominate prefill time for long interactions.

### 4.5 Agentic Context Sharing

In multi-agent systems, CMX acts as a **shared context bus**:

- Agent A (e.g., a research agent) writes its KV cache to CMX after processing a document corpus.
- Agent B (e.g., a summarization agent) reads Agent A's KV cache from CMX to build on the existing context without reprocessing the same documents.
- Agent C (e.g., a verification agent) reads both Agent A's and Agent B's caches to cross-check findings.

This pattern eliminates redundant prefill computation across agents, reducing total GPU-hours per agentic task. CMX's hash-based addressing enables fine-grained sharing --- agents can share individual KV blocks (corresponding to specific document segments) rather than entire monolithic caches.

### 4.6 Lifecycle Management and Garbage Collection

KV cache blocks in CMX are inherently ephemeral. Unlike traditional storage data that persists until explicitly deleted, KV blocks have natural expiration triggers:

- **Session timeout:** When a user session ends or idles beyond a configurable threshold, all associated KV blocks become candidates for eviction.
- **Context window sliding:** For models using sliding-window attention, KV blocks corresponding to tokens that have fallen outside the attention window can be garbage collected.
- **Reference counting:** When the last agent or session referencing a shared KV block terminates, the block's reference count drops to zero and it becomes reclaimable.
- **Capacity pressure:** When CMX flash utilization exceeds a high-water mark, the system evicts blocks based on a policy that considers recency of last access, block size, and sharing degree (widely-shared prefix blocks are retained longer than single-session blocks).

The BlueField-4's Grace CPU handles this lifecycle management locally, without involving the GPU compute nodes. This offloads metadata bookkeeping and garbage collection from the inference pipeline, ensuring that storage management overhead does not compete with token generation for GPU or host CPU cycles.

From an SSD wear perspective, the combination of high write rates and short block lifetimes means that CMX drives will experience significantly higher drive writes per day (DWPD) than typical enterprise storage workloads. NVIDIA has not published target DWPD specifications for CMX-certified drives, but given the write patterns, drives rated for 3+ DWPD (typical of mixed-use enterprise SSDs) are a reasonable starting point. SSD vendors participating in the STX ecosystem --- including Solidigm, Samsung, and Kioxia --- are expected to offer CMX-optimized firmware profiles that tune garbage collection for the ephemeral-block access pattern.

---

## 5. Performance Claims

### 5.1 Headline Numbers

NVIDIA claims the following for CMX/STX compared to traditional CPU-based storage architectures:

| Metric | CMX/STX Claim | Comparison Basis |
|---|---|---|
| Token throughput | **Up to 5x higher tokens/second** | vs. traditional storage approaches |
| Power efficiency | **Up to 5x better tokens/watt** | vs. traditional storage approaches |
| Energy efficiency | **4x better** | vs. CPU-based storage (STX overall) |
| Data ingestion | **2x faster** | vs. CPU-based storage (STX overall) |
| Time-to-first-token | **Up to 14x--20x faster** | vs. recomputing KV cache from scratch |

Sources: [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption), [NVIDIA Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/), [VAST Data](https://www.vastdata.com/blog/how-nvidia-dynamo-vast-unlock-context-reuse-at-scale)

### 5.2 Methodology and Caveats

NVIDIA's published materials compare CMX against "traditional storage approaches" and "CPU-based storage architectures" without specifying the exact baseline configuration (specific CPU generation, SSD model, network fabric, or software stack). This makes independent verification difficult. However, the claims are architecturally plausible for the following reasons:

**Token throughput (5x).** The primary mechanism is **elimination of GPU decode stalls**. When KV cache must be fetched from general-purpose storage over a non-optimized path, the GPU idles during the transfer. CMX's combination of RDMA transport, BlueField-4 hardware offload, and predictive prefetching is designed to keep KV blocks available in G1 before the decode step needs them. Even modest reductions in stall frequency compound into large throughput gains because the decode phase is inherently sequential --- one stall delays all subsequent tokens.

**Power efficiency (5x).** This follows directly from throughput improvement. If GPUs spend less time idle (waiting for KV data) and less time recomputing previously generated context, the same number of tokens is generated with fewer GPU-hours. Since GPUs dominate power consumption in inference pods, reducing GPU idle time translates almost linearly into better tokens-per-watt. Additionally, BlueField-4's hardware-accelerated data path (encryption, CRC, NVMe management) consumes far less power than equivalent CPU-based processing on general-purpose x86 storage controllers.

**TTFT improvement (14x--20x).** This is the most straightforward claim. Fetching a pre-computed KV cache from storage (even at NVMe latencies) is dramatically faster than rerunning the prefill computation on the GPU. For a 128K-token context, prefill can take seconds on a single GPU; fetching the corresponding KV cache from CMX takes milliseconds.

**What the claims do not address:** The 5x figures are likely measured under conditions that maximize CMX's advantage --- long context lengths, high KV cache reuse rates, and workloads where the baseline system experiences significant stalls. For short-context, single-turn workloads where KV cache fits entirely in HBM, CMX provides no throughput benefit (though it may still improve multi-user scaling by freeing HBM for additional concurrent sessions).

---

## 6. CMX vs. Alternatives

| Dimension | GPU HBM Only | CPU DRAM Offload | CXL-Attached Memory | General-Purpose Shared Storage (G4) | **NVIDIA CMX (G3.5)** |
|---|---|---|---|---|---|
| **Scope** | Per-GPU | Per-node | Per-rack (emerging) | Cluster-wide | **Pod-level** |
| **Capacity** | 192 GB/GPU | 1--2 TB/node | 10s TB/rack | Petabytes | **Petabytes/pod** |
| **Latency** | ~ns | ~us (PCIe) | ~100--300 ns (CXL) | ~ms | **~100--500 us** |
| **Bandwidth** | TB/s (HBM) | ~100 GB/s (PCIe) | ~1.5 TB/s (CXL 4.0) | ~10s GB/s | **800 Gbps/node (100 GB/s)** |
| **Sharing** | None | None | Limited (rack) | Full cluster | **Full pod** |
| **KV-aware** | Via software | Via software | Via software | No | **Hardware + software** |
| **Data protection** | None | None | None | Enterprise-grade | **Line-rate CRC + encryption** |
| **Production readiness** | Now | Now | Pre-production | Now | **H2 2026** |
| **Cost/GB** | $$$$$ | $$$ | $$$$ | $ | **$$** |

**CXL deserves special attention** as the most direct architectural competitor to CMX for KV cache offload. CXL offers lower latency (sub-microsecond) and cache-coherent access, but it requires CXL-capable CPUs, GPUs, and switches that are not yet broadly deployed. CXL 4.0's memory pooling capabilities are compelling on paper, but the ecosystem is 1--2 years behind CMX in production readiness. CMX's use of standard Ethernet (via Spectrum-X and RoCE) is a pragmatic choice that leverages existing data center fabric investments.

**General-purpose storage vendors** (VAST Data, WEKA, Dell, NetApp) are not displaced by CMX --- they continue to serve the G4 tier. Several are actively integrating with CMX and Dynamo. VAST Data's open-source VUA (VAST Undivided Attention) software, WEKA's Augmented Memory Grid, and Dell's Project Lightning all provide KV cache offload capabilities that can complement or compete with CMX depending on the deployment topology. For detailed analysis of storage vendor KV cache strategies and SSD readiness, see [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md).

---

## 7. Use Cases

### 7.1 Long-Context Inference

The most immediate use case for CMX is serving models with 128K--1M+ token context windows. At these context lengths, KV cache per request consumes 15--40+ GB of HBM, severely limiting concurrency. CMX allows operators to evict completed prefill KV blocks from HBM to the G3.5 tier, freeing capacity for additional concurrent sessions while retaining the ability to quickly retrieve context for ongoing decode.

**Storage engineer perspective:** The I/O pattern is characterized by large sequential writes (KV block eviction from HBM, typically 10s of MB per block) followed by large sequential reads (KV block retrieval for decode). Write-heavy phases occur during and immediately after prefill; read-heavy phases occur during decode. The working set size per session is proportional to context length. SSD endurance is a concern at scale --- KV blocks are ephemeral (minutes to hours of lifetime) with high write amplification if the SSD firmware is not optimized for this access pattern.

### 7.2 Multi-Turn Conversations

Production chat services (customer support, coding assistants, research tools) generate multi-turn sessions where each turn builds on the accumulated context. Without CMX, the KV cache from prior turns must either be held in GPU HBM (expensive at scale) or recomputed from the conversation transcript (slow, wasteful). CMX retains the session's KV cache across turns, enabling sub-second continuation of long conversations.

**Storage engineer perspective:** This workload resembles a session-affinity cache with TTLs measured in minutes to hours. The access pattern is write-once-read-many within a session, with the full session evicted after timeout. Capacity planning must account for peak concurrent sessions multiplied by average session KV cache size.

### 7.3 Agentic AI Workflows

Agentic AI systems --- where an orchestrator dispatches specialized sub-agents that reason, plan, and act in multi-step loops --- represent the highest-value CMX use case. Each agent step generates KV cache that subsequent steps may need. Without shared context storage, each agent must either retain all prior context in HBM (impractical for long chains) or re-ingest prior outputs as text (losing the computational investment in KV generation).

CMX enables agents to **write KV blocks to a shared namespace** and **read each other's cached context**, eliminating redundant computation. For a 10-step agentic workflow where each step processes 100K tokens of context, the difference between recomputing vs. reusing KV cache can be a 10x reduction in total GPU compute.

**Storage engineer perspective:** Agentic workloads generate complex dependency graphs of KV blocks with partial sharing. The storage system must support fine-grained block-level addressing (not just session-level), concurrent readers and writers, and garbage collection of orphaned blocks when agent workflows complete or fail.

### 7.4 Multi-User KV Reuse (Prefix Caching)

Many inference scenarios involve shared prefixes --- system prompts, RAG (retrieval-augmented generation) document contexts, or common instruction sets that are identical across users. CMX enables **pod-wide prefix caching**: the KV cache for a shared prefix is computed once and stored in CMX, then served to all subsequent requests that share that prefix.

For a RAG-heavy deployment serving 1,000 concurrent users against the same document corpus, prefix caching can eliminate >90% of prefill computation, directly translating to proportional reductions in GPU hours, power consumption, and time-to-first-token.

**Storage engineer perspective:** Prefix cache blocks are high-value, long-lived, and read-heavy. They should be pinned in BlueField-4's LPDDR5 staging buffer or on fast SSD tiers to minimize access latency. Deduplication is inherent in the hash-based addressing scheme --- identical prefixes map to the same KV block address.

---

## 8. Key Takeaways

1. **CMX is a storage architecture, not just a software feature.** It introduces a physically distinct tier (G3.5) in the AI inference memory hierarchy, with dedicated hardware (BlueField-4), dedicated networking (Spectrum-X RoCE), and dedicated software (DOCA Memos, NIXL). Storage engineers should treat it as a new infrastructure category alongside traditional SAN/NAS.

2. **The BlueField-4 DPU is the critical component.** Its 64-core Grace CPU, 800 Gbps networking, 128 GB LPDDR5 staging buffer, and hardware-accelerated CRC/encryption engines transform what would otherwise be commodity NVMe storage into a KV-cache-aware, inference-optimized storage node. The DPU handles the control plane (block lookup, placement, lifecycle management) and the data plane (NVMe I/O, RDMA transfers, integrity verification) without host CPU involvement.

3. **SSD selection matters.** CMX's performance depends on the NVMe drives behind BlueField-4. KV cache workloads are characterized by high write rates of ephemeral data (short block lifetimes), large sequential I/O, and high read throughput during decode phases. SSDs optimized for these patterns --- high endurance, low tail latency, high sequential throughput --- will outperform general-purpose drives. SSD vendors including Solidigm, Samsung, Micron, Kioxia, and SK hynix are actively co-designing drives for this workload. Detailed vendor analysis is available in [`reports/en/02-competitive-landscape.md`](../../en/02-competitive-landscape.md).

4. **Pod-level sharing is the differentiator over node-local SSD caching.** Any system can offload KV cache to local NVMe. CMX's value proposition is making that cache accessible to every GPU in the pod over a high-bandwidth Ethernet fabric. This enables prefix caching, multi-turn persistence, and agentic context sharing --- none of which are possible with node-local storage alone.

5. **The 5x performance claims are conditional.** They hold for workloads with long context, high KV reuse, and configurations where the alternative is general-purpose storage. Short-context, single-turn workloads that fit entirely in HBM will see minimal benefit. Operators should benchmark their specific workload mix against their existing infrastructure before committing to CMX deployment.

6. **CXL is the long-term architectural competitor.** CXL-attached memory pools offer lower latency and cache coherency that CMX cannot match. However, CXL infrastructure is 1--2 years behind CMX in production readiness. NVIDIA's bet is that Ethernet-attached flash (CMX) can be deployed at scale in H2 2026, while CXL memory pooling at pod scale remains a 2027--2028 story.

7. **Ecosystem alignment is broad.** Storage vendors (VAST, WEKA, Dell, NetApp, DDN, Hitachi, HPE, IBM), system OEMs (Supermicro, QCT, AIC), and cloud providers (CoreWeave, Crusoe, Lambda, OCI, Vultr, Nebius) have announced STX/CMX adoption plans. This breadth of support suggests CMX will become the de facto standard for inference context storage on NVIDIA platforms, at least until CXL alternatives mature. [NVIDIA Newsroom](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption)

8. **The software stack is as important as the hardware.** Dynamo's KV-aware routing, NIXL's heterogeneous transport abstraction, and DOCA Memos' key-value API collectively determine how effectively CMX hardware is utilized. Operators deploying CMX should plan for Dynamo integration as a prerequisite, not an afterthought.

9. **Capacity planning requires new models.** Traditional storage capacity planning based on dataset sizes and IOPS requirements does not apply to CMX. Instead, operators must model: peak concurrent sessions x average KV cache size per session x reuse factor x retention window. For example, a deployment serving 1,000 concurrent users with 128K-token contexts on a 70B model generates approximately 40 TB of active KV cache (1,000 x 40 GB). With a 50% reuse rate from prefix caching and a 30-minute retention window, the CMX tier needs roughly 20 TB of usable flash capacity plus headroom for write amplification and garbage collection overhead.

10. **The naming evolution signals NVIDIA's strategic intent.** CMX was initially announced as ICMS (Inference Context Memory Storage) at CES 2026, then rebranded to CMX at GTC 2026. The "Context Memory" framing deliberately positions this as a new category of infrastructure --- neither traditional storage nor traditional memory --- reinforcing NVIDIA's vision of an AI-native data center architecture where every tier is purpose-built for inference workloads.

---

*Report prepared March 2026. CMX platforms are expected from NVIDIA partners in H2 2026 as part of the Vera Rubin infrastructure rollout. All performance claims are based on NVIDIA's published materials and have not been independently verified at the time of writing.*
