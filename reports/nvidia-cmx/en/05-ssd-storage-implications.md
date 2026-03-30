# SSD and Storage Implications of NVIDIA CMX

**A deep technical analysis for storage engineers evaluating flash requirements in AI-native inference infrastructure**

> **Abstract.** NVIDIA's Context Memory Storage (CMX) platform, announced at GTC 2026 as the first implementation of the BlueField-4 STX architecture, creates an entirely new SSD workload category: AI-native context storage. By extending GPU memory hierarchies to flash-backed KV cache tiers, CMX imposes a distinct set of requirements on NVMe SSDs that differs materially from traditional data center storage workloads. This report analyzes those requirements across capacity, endurance, latency, and interface dimensions, evaluates QLC NAND fitness for CMX workloads, maps the SSD vendor ecosystem, and projects market impact through 2028. It is written for storage engineers and infrastructure architects planning procurement and deployment of CMX-aligned flash.

---

## 1. Introduction: CMX Creates a New SSD Workload Category

For the past decade, enterprise SSD workloads have been classified along a well-understood spectrum: write-intensive (databases, logging), mixed (virtualization, general compute), and read-intensive (CDN, analytics, model serving). NVIDIA's CMX platform introduces a workload that does not fit cleanly into any of these categories. It is best described as **ephemeral context storage** -- high-capacity, latency-sensitive, write-churning, but ultimately read-dominated in aggregate.

The technical driver is the KV cache. Large language model inference generates key-value pairs for each token in a conversation's context window. As context windows expand to millions of tokens and agentic AI systems maintain long-running, multi-turn sessions, KV cache footprints per session can reach multiple gigabytes. When the aggregate KV cache of concurrent sessions exceeds GPU HBM capacity, the system must spill context to lower tiers -- first to host DRAM, then to local NVMe SSDs, and finally to shared pod-level flash via CMX ([NVIDIA CMX Platform](https://www.nvidia.com/en-us/data-center/ai-storage/cmx/)).

NVIDIA's CMX provides this pod-level flash tier. Powered by BlueField-4 DPUs and Spectrum-X Ethernet, CMX manages NVMe SSDs as a shared, RDMA-accelerated context memory pool that can be accessed by any GPU in the pod. NVIDIA claims up to 5x improvement in tokens per second and 5x greater power efficiency versus traditional storage approaches ([NVIDIA BlueField-4 STX Announcement](https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption)). The software layer, NVIDIA Dynamo, handles KV-aware routing and placement, making the flash tier transparent to inference workloads ([NVIDIA Dynamo KVBM Design](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)).

For storage engineers, the implication is straightforward: every Vera Rubin inference pod will require a purpose-built flash tier, and the SSDs populating that tier face a workload profile unlike anything in current qualification playbooks.

---

## 2. CMX SSD Requirements

### 2.1 Capacity: TB-Scale per Node, PB-Scale per Pod

CMX operates at pod scale. A Vera Rubin NVL72 rack integrates 72 GPUs, each with HBM4 local memory. When KV cache spills beyond HBM and host DRAM, the CMX flash tier must absorb context for potentially thousands of concurrent inference sessions.

**Per-session KV cache sizing.** For a model like a 70B-parameter LLM with a 1M-token context window, each session's KV cache can consume approximately 2-4 GB (depending on quantization and model architecture). A 405B-parameter model at comparable context lengths can require 8-16 GB per session. At 1,000 concurrent sessions per pod, the aggregate KV cache footprint reaches 2-16 TB of active context data -- and this is before accounting for retained context from recently completed sessions that may be reused.

**Pod-level capacity.** NVIDIA describes CMX as providing "petabytes of shared capacity per GPU pod" ([NVIDIA Developer Blog](https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/)). In practice, early CMX reference designs from partners like Supermicro and AIC populate storage nodes with high-density NVMe drives. A CMX storage node equipped with 24x 25.6 TB SSDs provides approximately 614 TB of raw flash capacity. A pod with 4-8 such storage nodes reaches 2.5-5 PB.

**Implication for SSD selection.** Capacity per drive matters. Drives at 15.36 TB are viable but require more slots, increasing cabling and power overhead. The sweet spot for CMX is likely 25.6-61.44 TB per drive, with 122 TB+ drives becoming attractive as they reach production maturity.

### 2.2 Read Latency: Sub-100 Microseconds at the Drive

CMX's value proposition depends on serving cached KV blocks back to GPUs faster than recomputing them. The latency budget is tight:

- **GPU HBM access:** ~1 us
- **Host DRAM access:** ~0.1-0.5 us (local), ~1-5 us (CXL-attached)
- **CMX flash tier target:** 10-80 us at the drive, plus network traversal
- **Recomputation cost:** Milliseconds to hundreds of milliseconds depending on sequence length

For the CMX flash tier to be useful, end-to-end latency (drive + BlueField-4 DPU + RDMA network + GPU ingress) must remain well under the recomputation penalty. With RDMA network traversal adding 5-15 us and DPU processing overhead of a few microseconds, the SSD itself must deliver reads in the **sub-80 us range** consistently, not just at the median but at the 99th percentile.

This is within reach of current enterprise NVMe SSDs for sequential reads of the 33 MB KV cache blocks that characterize this workload. However, QoS consistency matters more than peak throughput. Tail latency spikes from garbage collection, SLC-to-QLC folding, or read-retry operations could cause GPU stalls. Drives deployed in CMX must exhibit deterministic latency behavior under sustained mixed workloads.

### 2.3 Write Endurance: Ephemeral High-Churn Context

KV cache is ephemeral by nature. Context blocks are written when sessions are created or extended, and evicted when sessions end or when the cache is pressured. The write pattern is:

- **Large sequential writes:** KV blocks are typically written as contiguous 2-33 MB chunks (varying by framework and quantization).
- **High churn on active data:** Sessions lasting minutes to hours generate context that is written once, potentially read many times, then discarded.
- **Frequency-gated disk offload:** NVIDIA Dynamo's KVBM (KV Block Manager) implements disk offload filtering by default, only offloading KV blocks with a reuse frequency of 2 or more to SSDs. This is an explicit endurance protection mechanism ([NVIDIA Dynamo KVBM Documentation](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)).

**Estimating write volume.** Consider a CMX node with 614 TB of flash serving a pod with 1,000 concurrent sessions, each generating 4 GB of KV cache with an average session lifetime of 30 minutes. If 50% of sessions qualify for disk offload (frequency >= 2), that produces:

- 500 sessions x 4 GB = 2 TB of new writes per 30-minute cycle
- 96 TB of writes per day across the node
- Across 24x 25.6 TB drives = 4 TB/day per drive
- DWPD = 4 / 25.6 = ~0.16 DWPD

This estimate is conservative. Under aggressive agentic workloads with rapid session creation and longer contexts, write volumes could reach 0.5-1.0 DWPD per drive. The write pattern is heavily sequential (favorable for NAND endurance), but the churn is continuous.

**DWPD requirement range:** 0.3-1.0 DWPD appears to be the design envelope, depending on workload intensity. This sits at the boundary between read-intensive QLC (0.3-0.58 DWPD) and mainstream TLC (1-3 DWPD).

### 2.4 Access Patterns: Sequential Writes, Mixed Reads

Research characterizing KV cache I/O patterns at the device level reveals that approximately 78% of access is sequential reads, with each inference service issuing dedicated sequential reads for its assigned KV cache blocks ([An I/O Characterizing Study of Offloading LLM Models and KV Caches to NVMe SSD](https://atlarge-research.com/pdfs/2025-cheops-llm.pdf)). The Linux block layer segments each 33 MB KV cache block into sub-requests of primarily 2.8 MB and 6.1 MB, and because these request streams overlap in time from multiple concurrent services, the NVMe device receives multiple concurrent sequential-read sequences.

This means the SSD's ability to detect and maintain sequential access patterns across interleaved streams is critical. Drives with aggressive read-ahead and multi-stream awareness will outperform those optimized purely for random 4K IOPS.

### 2.5 Interface: PCIe Gen5 Today, Gen6 Imminent

The Vera Rubin platform supports PCIe Gen6 and CXL 3.1 ([NVIDIA Vera Rubin Platform](https://developer.nvidia.com/blog/inside-the-nvidia-rubin-platform-six-new-chips-one-ai-supercomputer/)). Early CMX deployments in H2 2026 will use PCIe Gen5 NVMe SSDs, as these are what is qualified and available at scale today. Micron's 9650 -- the world's first PCIe Gen6 data center SSD, in mass production since February 2026 -- delivers 28 GB/s sequential reads and is explicitly positioned for BlueField-4 STX workloads ([Micron 9650 for Vera Rubin](https://investors.micron.com/news-releases/news-release-details/micron-high-volume-production-hbm4-designed-nvidia-vera-rubin)).

For a CMX storage node serving aggregate throughput of 37+ GB/s to the pod (as observed in testing environments with multiple concurrent users), PCIe Gen5 x4 (~14 GB/s per drive) may suffice with enough drives in parallel. PCIe Gen6 x4 (~28 GB/s per drive) halves the drive count needed for the same aggregate bandwidth, improving density and power efficiency.

**Form factors.** CMX storage nodes use EDSFF form factors. The KIOXIA CM9 targets E3.S at 25.6 TB; the Micron 9650 ships in E1.S. E3.S is the preferred form factor for high-capacity CMX nodes due to its thermal headroom and capacity density, though E1.S remains relevant for bandwidth-optimized configurations. As the [Technology Roadmap](../../en/05-technology-roadmap-2025-2030.md) documents, E3.S is projected to exceed 35% market share by 2029 and is already dominant for hyperscale by 2026.

---

## 3. QLC SSD Fit for CMX

### 3.1 Read-Heavy Retrieval Matches QLC's Strength

The fundamental I/O characteristic of CMX -- write once (or a few times), read many -- aligns with the workload profile that QLC NAND was engineered for. As detailed in [QLC Technology Fundamentals](../../en/01-qlc-technology-fundamentals.md), QLC read latency (tR) of 75-120 us is only marginally slower than TLC's 50-75 us. For the large sequential reads (2.8-33 MB blocks) that dominate KV cache retrieval, the per-page read latency difference is amortized across thousands of pages, making aggregate throughput nearly equivalent.

QLC's read performance advantage over TLC is economic: for the same dollar spend, QLC delivers roughly 33% more capacity. In a CMX pod requiring petabytes of flash, this translates to hundreds of thousands of dollars in savings per pod.

### 3.2 Endurance Analysis for Ephemeral KV Cache

The endurance question is nuanced. Current enterprise QLC SSDs are rated at 0.3-0.58 DWPD (e.g., Solidigm D5-P5336 at 0.58 DWPD over 5 years, as documented in [QLC Technology Fundamentals Section 4.4](../../en/01-qlc-technology-fundamentals.md)). Our analysis in Section 2.3 estimates CMX write volumes of 0.16-1.0 DWPD depending on workload intensity.

**Where QLC fits:** For inference workloads with moderate context churn -- standard chatbot serving, document Q&A, summarization -- the estimated 0.16-0.3 DWPD is comfortably within QLC's endurance envelope. Dynamo's frequency-gated offload filtering (only persisting blocks reused 2+ times) further reduces writes to the flash tier, acting as a software-level endurance protector.

**Where TLC is needed:** Aggressive agentic workloads -- multi-agent orchestration with rapid session creation/teardown, chain-of-thought reasoning with frequent context updates -- can push write volumes toward 0.5-1.0 DWPD. At the high end, this exceeds QLC endurance ratings. TLC drives rated at 1-3 DWPD provide headroom for these scenarios.

The KIOXIA CM9, which KIOXIA developed specifically for CMX KV cache workloads, is a TLC drive with 3 DWPD endurance at 25.6 TB in E3.S -- a deliberate choice reflecting the reality that worst-case CMX write churn exceeds QLC limits ([KIOXIA CM9 for CMX](https://www.businesswire.com/news/home/20260316056420/en/KIOXIA-Announces-New-SSD-Model-Optimized-for-AI-GPU-Initiated-Workloads)). Similarly, Samsung's PM1753 -- its TLC SSD designed for the CMX platform -- delivers 14.5 GB/s sequential throughput with write endurance engineered for high-churn AI patterns ([Samsung PM1753 for CMX](https://esaitech.com/blogs/insights/samsung-pm1753-ssd-ai-and-data-center-storage)).

### 3.3 SLC Caching Relevance

SLC caching, as analyzed in [QLC Technology Fundamentals Section 6](../../en/01-qlc-technology-fundamentals.md), is particularly relevant for CMX. KV cache writes are bursty: a wave of new sessions or context extensions can produce a surge of writes that must be absorbed quickly. The SLC cache absorbs these bursts in single-bit-per-cell mode at native NAND write speeds, then folds to QLC during quieter periods.

For QLC in CMX, two properties matter:

1. **Dynamic SLC cache sizing.** Because CMX drives may not be filled to capacity (KV cache is ephemeral and constantly recycled), a substantial portion of unused QLC blocks can serve as dynamic SLC cache. A 61.44 TB QLC drive at 60% fill level could maintain hundreds of gigabytes of SLC cache, comfortably absorbing write bursts.

2. **Folding latency impact.** SLC-to-QLC folding is a background operation that competes for NAND bandwidth. During folding, read latency can spike. For CMX's latency-sensitive KV retrieval, this is a concern. Drives with intelligent folding schedulers that defer folding during active read periods will perform better in CMX deployments.

SanDisk's UltraQLC platform, which bypasses SLC caching entirely with Direct Write QLC technology ([SanDisk UltraQLC at FMS 2025](https://www.sandisk.com/company/newsroom/press-releases/2025/2025-08-05-sandisk-showcases-ultraqlc-technology-platform-with-milestone-enterprise-ssd-capacity-at-fms-2025)), could prove advantageous for CMX by eliminating folding-induced latency spikes, though at the cost of raw write throughput.

### 3.4 Capacity/Cost Advantage

The economic case for QLC in CMX is compelling at scale:

| Drive | Technology | Capacity | Approx. $/GB | Cost for 614 TB Node |
|-------|-----------|----------|--------------|---------------------|
| Solidigm D5-P5336 | QLC Gen4 | 122.88 TB | ~$0.10 | ~$61,400 (5 drives) |
| KIOXIA CM9 | TLC Gen5 | 25.6 TB | ~$0.18-0.22 | ~$110,000-$135,000 (24 drives) |
| Micron 9650 | TLC Gen6 | 30.72 TB | ~$0.20-0.25 | ~$123,000-$154,000 (20 drives) |

QLC provides a roughly 50-60% cost reduction for the flash tier. For a pod with 4-8 CMX storage nodes, QLC saves $200,000-$750,000 in SSD procurement alone. The question is whether the endurance budget and latency QoS are sufficient for the specific workload profile.

### 3.5 Practical Recommendation: Tiered CMX Flash

The optimal CMX flash strategy for most deployments is a tiered approach within the CMX tier itself:

- **Hot context tier (TLC):** High-endurance, low-latency TLC drives (KIOXIA CM9, Samsung PM1753) hold actively churning KV cache with the highest reuse frequency. Sized at 20-30% of total CMX capacity.
- **Warm context tier (QLC):** High-capacity QLC drives (Solidigm D5-P5336, Micron 6600 ION) hold retained context that is accessed less frequently but must remain available for multi-turn sessions. Sized at 70-80% of total CMX capacity.

NVIDIA Dynamo's KV Block Manager already implements tiering policies including hot block promotion and cold block demotion ([NVIDIA Dynamo KVBM](https://docs.nvidia.com/dynamo/design-docs/component-design/kvbm-design)), making this architecture software-ready.

---

## 4. STX Platform: The Broader Storage Architecture

### 4.1 STX Architecture and Purpose

STX (Storage Technology eXtension) is NVIDIA's modular reference architecture for AI-native storage, built around BlueField-4 DPUs and the Vera Rubin platform. Announced at GTC 2026, STX is broader than CMX -- it defines how accelerated storage infrastructure should be built for the entire AI lifecycle ([NVIDIA STX Architecture](https://www.nvidia.com/en-us/data-center/ai-storage/stx/)).

STX leverages three core components:

- **BlueField-4 DPU:** A storage-optimized DPU that manages NVMe SSDs, runs storage services, and offloads data integrity, encryption, and compression. BlueField-4 replaces the conventional CPU in the storage I/O path, providing higher throughput per watt.
- **ConnectX-9 SuperNIC:** Provides high-bandwidth RDMA connectivity between compute and storage nodes.
- **Spectrum-X Ethernet:** The network fabric with advanced congestion control, adaptive routing, and lossless RoCE for deterministic latency.

### 4.2 Difference from CMX

CMX is the first rack-scale implementation of the STX architecture, optimized specifically for KV cache context memory. STX is the general-purpose framework:

| Dimension | CMX | STX (General) |
|-----------|-----|---------------|
| Primary workload | KV cache for inference | Training data, checkpoints, model repos, RAG |
| Data persistence | Ephemeral (hours-days) | Persistent (weeks-years) |
| Access pattern | Large sequential R/W, RDMA | Mixed random + sequential |
| SSD priority | Latency QoS, capacity | Throughput, endurance |
| Typical SSD | TLC/QLC NVMe, 25-122 TB | TLC NVMe, 3.84-30.72 TB |

### 4.3 STX Use Cases Beyond CMX

STX-based storage nodes serve multiple roles in the AI data center:

- **Training data staging:** Feeding training pipelines with shuffled dataset batches. Requires sustained sequential read bandwidth. High-capacity QLC SSDs are well-suited.
- **Checkpoint storage:** Periodic model state snapshots during training. Write-intensive bursts of hundreds of GB to TB. Requires moderate endurance (1-3 DWPD). TLC preferred.
- **Model repositories:** Storing and serving model weights for deployment. Read-dominant, capacity-sensitive. QLC is ideal.
- **RAG vector databases:** Retrieval-augmented generation requires low-latency access to embedding indices. Mixed read/write with emphasis on random read IOPS. TLC with high IOPS.

### 4.4 Industry Adoption

STX-based platforms are expected from partners in H2 2026. Confirmed design partners include DDN, Dell Technologies, HPE, IBM, NetApp, and VAST Data for storage systems, with Supermicro, AIC, and Quanta Cloud Technology as manufacturing partners. Supermicro has already unveiled a CMX storage server prototype combining NVIDIA Vera CPU with BlueField-4 ([Supermicro STX Server](https://ir.supermicro.com/news/news-details/2026/Supermicro-Among-First-to-Unveil-NVIDIA-BlueField-4-STX-Storage-Server-to-Improve-AI-Inference-Performance/default.aspx)).

---

## 5. SSD Vendor Readiness

### 5.1 CMX-Ready Products

The following SSD products have been explicitly demonstrated, announced, or confirmed for CMX/STX compatibility at GTC 2026:

**KIOXIA**
- **CM9 Series (TLC, PCIe 5.0, E3.S, 25.6 TB, 3 DWPD):** Purpose-built for CMX KV cache. 14.8 GB/s read, 11 GB/s write, 3.4M IOPS. Sampling Q3 2026 ([KIOXIA CM9 Announcement](https://www.businesswire.com/news/home/20260316056420/en/KIOXIA-Announces-New-SSD-Model-Optimized-for-AI-GPU-Initiated-Workloads)).
- **GP Series (XL-FLASH SLC/MLC, E1.S/E3.S, 800 GB SLC + 1,600 GB MLC):** Ultra-low-latency option using Storage Class Memory for the hottest KV cache data. 512-byte granularity access. Evaluation samples by end of 2026 ([KIOXIA GP Series](https://www.servethehome.com/kioxia-gp-series-and-cm9-launched-for-the-era-of-agentic-ai-storage/)).

**Micron**
- **9650 (TLC, PCIe Gen6, E1.S, up to 30.72 TB):** World's first Gen6 SSD, 28 GB/s reads, 5.5M random read IOPS. In mass production since February 2026. Explicitly optimized for BlueField-4 STX ([Micron 9650](https://www.micron.com/products/storage/ssd/data-center-ssd/9650-ssd)).
- **6600 ION (QLC, PCIe Gen5, E3.S/E3.L, 30.72-245 TB):** High-capacity option for warm context and model storage tiers. The 122 TB and 245 TB variants are in qualification at hyperscalers ([Micron 6600 ION](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)).

**Samsung**
- **PM1753 (TLC, PCIe Gen5, E3.S):** Confirmed for CMX platform supply. 14.5 GB/s sequential throughput, 3.3M random read IOPS. Low-power design optimized for inference workloads ([Samsung PM1753 for Vera Rubin](https://esaitech.com/blogs/insights/samsung-pm1753-ssd-ai-and-data-center-storage)).

**Solidigm**
- **D5-P5336 (QLC, PCIe Gen4, 2.5"/E3.S, up to 122.88 TB):** Demonstrated in AIC's CMX-aligned F2026 platform at GTC 2026. 0.58 DWPD endurance. Highest capacity PCIe SSD shipping today ([Solidigm D5-P5336](https://www.solidigm.com/products/data-center/d5/p5336.html)).
- **D5-P5436 (QLC, PCIe Gen5, next-gen):** Based on SK Hynix 321-layer QLC. Expected to extend capacity beyond 122 TB with Gen5 bandwidth.

### 5.2 PCIe Gen6 SSD Availability

As documented in the [Technology Roadmap Section 6.2](../../en/05-technology-roadmap-2025-2030.md), PCIe Gen6 doubles per-lane bandwidth to 64 GT/s using PAM-4 signaling. The Micron 9650 is the sole Gen6 SSD in mass production as of March 2026. Samsung and KIOXIA have not yet announced Gen6 SSD products, though both have Gen6 PHY development underway. Solidigm's Gen5-based D5-P5436 is the next expected product; a Gen6 variant is likely in the 2027-2028 timeframe.

For CMX deployments launching in H2 2026, the practical reality is PCIe Gen5 SSDs from multiple vendors, with the Micron 9650 as the sole Gen6 option. Gen6's bandwidth advantage (2x per drive) will become more valuable as CMX pod scale increases and per-node drive counts decrease.

### 5.3 Vendor-by-Vendor CMX Readiness Summary

| Vendor | CMX Product | NAND Type | Interface | Capacity | Endurance | Availability |
|--------|-------------|-----------|-----------|----------|-----------|-------------|
| KIOXIA | CM9 | TLC (BiCS8) | Gen5 | 25.6 TB | 3 DWPD | Q3 2026 (samples) |
| KIOXIA | GP Series | SLC/MLC (XL-FLASH) | Gen5 | 2.4 TB | High | End 2026 (eval) |
| Micron | 9650 | TLC (G9) | Gen6 | Up to 30.72 TB | 1 DWPD | Shipping |
| Micron | 6600 ION | QLC (G9) | Gen5 | 30.72-245 TB | 0.58 DWPD | Shipping/qual |
| Samsung | PM1753 | TLC (V8) | Gen5 | TBD | TBD | H2 2026 |
| Solidigm | D5-P5336 | QLC (192L) | Gen4 | Up to 122 TB | 0.58 DWPD | Shipping |
| Solidigm | D5-P5436 | QLC (321L) | Gen5 | >122 TB | TBD | 2026-2027 |

Cross-referencing the [Competitive Landscape](../../en/02-competitive-landscape.md): all five major NAND manufacturers (Samsung, SK Hynix/Solidigm, Micron, KIOXIA/SanDisk, YMTC) have products or roadmaps relevant to CMX, though YMTC's participation is limited to the Chinese domestic market due to US export controls.

---

## 6. Impact on the SSD Market

### 6.1 A New Demand Driver

CMX represents the first storage platform where SSDs are spec'd not for traditional enterprise storage, but as a GPU memory extension. This changes the procurement conversation:

- **Buyers** are GPU infrastructure teams, not storage teams. SSD selection criteria shift from $/IOPS and $/TB to **latency QoS, bandwidth density, and power efficiency per token throughput**.
- **Volumes** scale with GPU deployment, not with data growth. Every inference pod requires a flash tier regardless of the application's data size.
- **Refresh cycles** may be shorter than traditional 5-year enterprise SSD lifecycles, since KV cache writes -- even when filtered -- accumulate endurance wear continuously.

### 6.2 Estimated SSD Capacity per CMX Node and Pod

Based on announced reference architectures and partner designs:

| Configuration | Drives per Node | Capacity per Node | Nodes per Pod | Total Flash per Pod |
|---------------|----------------|-------------------|---------------|-------------------|
| High-capacity (QLC) | 5x 122 TB | ~614 TB | 4-8 | 2.5-5 PB |
| Balanced (TLC) | 24x 25.6 TB | ~614 TB | 4-8 | 2.5-5 PB |
| Bandwidth-opt (Gen6) | 20x 30.72 TB | ~614 TB | 4-8 | 2.5-5 PB |
| Tiered (QLC + TLC) | 5x 122 TB + 8x 25.6 TB | ~815 TB | 4-8 | 3.3-6.5 PB |

At even modest adoption -- 10,000 inference pods deployed globally by end of 2027 -- CMX would consume 25-50 EB of enterprise SSD capacity. For reference, total enterprise SSD shipments reached approximately 120 EB in 2025 ([TrendForce Q3 2025 Enterprise SSD](https://www.trendforce.com/presscenter/news/20251205-12819.html)). CMX alone could represent 20-40% of incremental enterprise SSD demand growth.

### 6.3 QLC TAM Expansion

CMX expands the total addressable market for QLC SSDs in two ways:

1. **Direct CMX deployment.** QLC drives serving the warm context tier represent net-new demand that did not exist before CMX. These are not HDD replacements (the traditional QLC TAM thesis documented in [Competitive Landscape Section 2.2](../../en/02-competitive-landscape.md)) -- they are memory-tier extensions.

2. **Cascade displacement.** As TLC SSDs are allocated to CMX hot-context tiers, the storage workloads those TLC drives previously served (training data staging, model repos, analytics) shift to QLC. This creates secondary QLC demand independent of CMX itself.

The combined effect is bullish for QLC vendors, particularly Solidigm (whose entire strategy is QLC-focused) and Micron (whose CuA architecture provides a structural cost advantage in QLC, as detailed in [Competitive Landscape Section 2.3](../../en/02-competitive-landscape.md)).

### 6.4 Procurement Pattern Changes

CMX changes how SSDs are procured for AI infrastructure:

- **Co-qualification with DPU/NIC stack.** SSDs must be validated against BlueField-4 DPU firmware, Dynamo software, and Spectrum-X networking. This adds qualification cycles beyond traditional server-level SSD qualification.
- **Power envelope awareness.** CMX storage nodes have strict power budgets. SSD power consumption (both active and idle) becomes a first-order selection criterion. Samsung's PM1753 explicitly positions on low power; KIOXIA's GP Series uses XL-FLASH partly for power efficiency.
- **Endurance monitoring integration.** Dynamo's disk offload filtering dynamically adjusts write behavior based on SSD health. SSDs that expose detailed SMART telemetry (remaining endurance, WAF, temperature) through standardized NVMe health logs enable better Dynamo optimization.

---

## 7. Future Directions

### 7.1 CXL-Attached Memory as Complement

CXL Type 3 memory expanders represent a potential intermediate tier between host DRAM and CMX flash. As the [Technology Roadmap Section 4](../../en/05-technology-roadmap-2025-2030.md) details, NAND-backed CXL devices present byte-addressable memory interfaces with microsecond-class latency -- roughly 10-50x slower than DRAM but potentially 10-100x faster than NVMe for small random accesses.

In a CMX-equipped pod, a CXL memory tier could serve as a "G2.5" layer:

| Tier | Medium | Latency | Capacity per Node | Role |
|------|--------|---------|-------------------|------|
| G1 | GPU HBM4 | ~1 us | 288 GB | Active KV cache |
| G2 | Host DRAM | ~0.1-0.5 us | 1-2 TB | Hot overflow |
| G2.5 | CXL NAND-backed | 1-10 us | 2-8 TB | Warm overflow (future) |
| G3 | Local NVMe SSD | 10-80 us | Hundreds of TB | Warm context |
| G3.5 | CMX flash (pod) | 20-100 us | Petabytes | Shared context |

The CXL 4.0 specification, released November 2025, provides the bandwidth (128 GT/s) and fabric-attached memory semantics needed for this tier. Samsung's CMM-D 2.0 CXL memory modules (128-256 GB, CXL 2.0) are already sampling. However, practical deployment of NAND-backed CXL in CMX environments is a 2028+ prospect, contingent on CPU platform support and software stack maturity.

### 7.2 Computational Storage for CMX

Computational Storage Devices (CSDs) could add value in CMX by performing KV cache compression or decompression at the drive level, reducing RDMA bandwidth consumption and increasing effective flash capacity. ScaleFlux's transparent compression approach, as discussed in the [Technology Roadmap Section 7](../../en/05-technology-roadmap-2025-2030.md), has demonstrated 2-4x effective capacity gains. Applied to a QLC CMX tier, this could stretch a 614 TB node to 1.2-2.4 PB effective capacity.

The barrier is integration complexity: CMX's software stack (Dynamo, BlueField-4 DOCA) would need to coordinate with CSD firmware for compression/decompression, adding latency uncertainty. This is a 2028+ opportunity.

### 7.3 FDP and ZNS for CMX Write Amplification

FDP (Flexible Data Placement) and ZNS (Zoned Namespaces) are directly relevant to CMX SSD endurance. As documented in the [Technology Roadmap Section 5](../../en/05-technology-roadmap-2025-2030.md):

- **FDP** allows Dynamo to tag KV cache writes with placement hints (Reclaim Unit Handles), separating short-lived context blocks from longer-lived ones. This reduces write amplification from 3-5x to 1.2-1.5x, effectively doubling or tripling QLC endurance.
- **ZNS** provides even tighter WAF reduction (near 1.0x) but requires sequential write ordering that may conflict with CMX's bursty, multi-stream write patterns.

FDP is the more practical path for CMX. If FDP-capable firmware becomes standard on CMX-qualified SSDs -- which the [Technology Roadmap](../../en/05-technology-roadmap-2025-2030.md) projects by 2027-2028 -- the endurance gap between QLC and TLC narrows significantly. A QLC drive with FDP achieving WAF of 1.3x could deliver effective endurance equivalent to a non-FDP QLC drive rated at 1.5-2x higher DWPD.

This is transformative for QLC's role in CMX. With FDP, the 0.58 DWPD QLC drives could sustain effective write volumes of 0.75-0.85 DWPD -- covering all but the most extreme agentic workloads.

---

## 8. Key Takeaways

1. **CMX creates genuine new SSD demand.** Every Vera Rubin inference pod requires petabytes of NVMe flash for KV cache context storage. This is not a replacement of existing storage; it is a new tier that did not exist before 2026.

2. **The CMX SSD profile is unique.** Large sequential writes, read-dominated retrieval, ephemeral data with continuous churn, and strict latency QoS requirements. No existing SSD workload category matches precisely.

3. **QLC is viable for CMX, but not universally.** QLC's cost and capacity advantages are compelling for the warm context tier (70-80% of CMX capacity). However, the hot context tier -- subject to aggressive write churn from agentic workloads -- requires TLC endurance. A tiered QLC+TLC strategy within CMX is the pragmatic optimum.

4. **Dynamo's frequency-gated offload is critical for QLC.** By filtering writes to only persist KV blocks with reuse frequency >= 2, Dynamo acts as an endurance protector. Storage engineers should validate that Dynamo's offload policies are properly tuned for their workload's session characteristics.

5. **FDP will be a QLC enabler for CMX.** When FDP-capable firmware becomes standard on CMX-qualified QLC SSDs (expected 2027-2028), write amplification reduction will significantly expand QLC's addressable share of CMX capacity.

6. **PCIe Gen6 matters for CMX density.** The Micron 9650 (Gen6, 28 GB/s) halves the drive count needed per node versus Gen5. As Gen6 products proliferate in 2027, CMX storage nodes will become more compact and power-efficient.

7. **Vendor differentiation is real.** KIOXIA's CM9 and GP Series, Micron's 9650 and 6600 ION, Samsung's PM1753, and Solidigm's D5-P5336/P5436 each occupy distinct positions in the CMX storage hierarchy. No single drive covers the full requirement space. Storage engineers must evaluate across the latency-capacity-endurance trade-off surface specific to their inference workload profile.

8. **SSD procurement for AI inference is now a GPU infrastructure decision.** CMX SSDs are co-qualified with BlueField-4, Dynamo, and Spectrum-X. Storage engineering teams must collaborate closely with GPU infrastructure teams to align SSD selection with inference serving requirements.

---

*Report current through March 2026. Based on publicly available specifications, vendor announcements at NVIDIA GTC 2026, and analysis of existing QLC SSD research in this repository.*
