# NVIDIA Vera Rubin CMX Research Report Suite — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a 5-document research suite on NVIDIA Vera Rubin pod architecture and CMX Context Memory Platform, in both English and Chinese, with cross-references to existing QLC SSD research.

**Architecture:** Each report is independently researched and written. Web research comes first, then synthesis into a structured markdown document, then Chinese translation. Report 1 (pod overview) provides context for all others, so it is written first. Reports 2-4 can be parallelized. Report 5 (SSD implications) depends on 1-4 and the existing QLC reports.

**Tech Stack:** Markdown documents, web research (WebSearch/WebFetch), git for version control.

---

## Task 1: Research & Write — Vera Rubin Pod Architecture Overview

**Files:**
- Create: `reports/nvidia-cmx/en/01-vera-rubin-pod-architecture.md`

**Research queries to execute:**

- "NVIDIA Vera Rubin pod architecture seven chips five rack-scale systems"
- "NVIDIA Vera CPU specifications Arm Neoverse DDR6 PCIe Gen6"
- "NVIDIA Rubin Ultra GPU HBM4e NVLink 6 specifications"
- "NVIDIA Vera Rubin pod DGX MGX CMX STX NetX"
- "NVIDIA Rubin GPU R100 specifications 2026"
- "NVIDIA NVLink 6 switch 130TB bandwidth specifications"
- "NVIDIA Vera Rubin vs Blackwell comparison specs"
- "NVIDIA GTC 2025 Vera Rubin announcement details"

For promising results, use WebFetch on NVIDIA developer blogs, newsroom, and SemiAnalysis.

**Report sections to cover:**

1. **Introduction** — Vera Rubin as NVIDIA's next-generation AI supercomputer platform, why it matters, timeline
2. **The Seven Chips**
   - Vera CPU — 86 Arm Neoverse V3 cores, 12-channel DDR6, PCIe Gen6 x16 lanes, CXL support
   - Rubin Ultra GPU — 2-die design, 16 HBM4e stacks, NVLink 6, FP4/FP8 performance
   - Rubin GPU (R100) — single-die variant, specs, positioning
   - BlueField-4 DPU — storage processor role, networking offload (details in Report 3)
   - NVLink 6 Switch — 130 TB/s aggregate, enabling multi-rack fabrics
   - CX9 SuperNIC — network interface for Spectrum-X
   - Spectrum-X Switch — Ethernet switch for east-west traffic
3. **The Five Rack-Scale Systems**
   - DGX — GPU compute nodes
   - MGX — modular GPU configurations
   - CMX — context memory storage (overview, details in Report 2)
   - STX — general-purpose AI storage
   - NetX — networking fabric
4. **Memory & Bandwidth Hierarchy** — HBM4e → NVLink 6 → Spectrum-X/RDMA → CMX (NVMe SSD), bandwidth at each tier
5. **Comparison to Previous Generations** — Blackwell → Vera Rubin key improvements, generational leaps
6. **Key Takeaways** — What this means for AI infrastructure planning

- [ ] **Step 1: Web research — Vera Rubin pod overview and chip specs**

Search for: Vera Rubin announcement details, chip specifications, pod composition.
Collect: specific specs (core counts, bandwidth numbers, memory capacities), architecture diagrams described in text.

- [ ] **Step 2: Web research — rack-scale systems and memory hierarchy**

Search for: DGX/MGX/CMX/STX/NetX details, memory hierarchy, bandwidth tiers.
Collect: system compositions, bandwidth numbers at each tier, node counts per rack.

- [ ] **Step 3: Web research — comparison to Blackwell and timeline**

Search for: Vera Rubin vs Blackwell specs, availability timeline, GTC announcements.
Collect: generational comparison data, confirmed dates, partner availability.

- [ ] **Step 4: Write the full report**

Synthesize into `reports/nvidia-cmx/en/01-vera-rubin-pod-architecture.md`.
Target: 4000-6000 words with inline source citations.

- [ ] **Step 5: Commit**

```bash
git add reports/nvidia-cmx/en/01-vera-rubin-pod-architecture.md
git commit -m "docs: add Vera Rubin pod architecture overview report (EN)"
```

---

## Task 2: Research & Write — CMX Context Memory Platform Deep Dive

**Files:**
- Create: `reports/nvidia-cmx/en/02-cmx-context-memory-platform.md`

**Research queries to execute:**

- "NVIDIA CMX Context Memory Platform architecture details"
- "NVIDIA CMX KV cache storage pod-level context tier"
- "NVIDIA CMX 5x tokens per second power efficiency claims"
- "NVIDIA DOCA Memos KV cache API CMX"
- "NVIDIA CMX agentic AI inference context sharing"
- "KV cache offloading SSD NVMe AI inference 2025 2026"
- "NVIDIA CMX vs traditional KV cache GPU memory offload"
- "NVIDIA CMX BlueField-4 NVMe SSD context memory"
- "NVIDIA STX CMX infrastructure agentic AI context storage"
- "AI inference KV cache bottleneck long context million tokens"

For promising results, use WebFetch on NVIDIA CMX product page, NAND Research, Blocks and Files, developer blogs.

**Report sections to cover:**

1. **Introduction** — The KV cache problem: why AI inference needs a new memory tier
2. **The KV Cache Challenge**
   - What KV cache is and why it grows with context length
   - Memory cost of million-token contexts (specific GB/TB numbers)
   - Current approaches: GPU HBM only, CPU DRAM offload, limitations of each
3. **CMX Architecture**
   - CMX as AI-native context tier — ephemeral, shareable, pod-level
   - CMX node composition: BlueField-4 + NVMe SSDs + Spectrum-X connectivity
   - How KV cache is stored, indexed, and retrieved
   - Pod-level context sharing across GPUs, sessions, and agents
4. **Data Flow**
   - Request arrives → KV cache lookup → cache hit (CMX retrieval) vs miss (recompute)
   - Write path: new KV cache entries from GPU → CMX storage
   - Multi-turn conversation: context persistence across turns
   - Agentic AI: context sharing between multiple agents
5. **Performance Claims**
   - 5x tokens/second improvement — how and why
   - 5x power efficiency — what's being compared
   - Latency characteristics: CMX retrieval vs recomputation
6. **CMX vs Alternatives**
   - GPU-only KV cache (limited by HBM capacity)
   - CPU DRAM offload (limited by PCIe bandwidth, CPU overhead)
   - CXL-attached memory (emerging, higher cost)
   - CMX (NVMe + BlueField-4 + RDMA — capacity, efficiency, cost)
7. **Use Cases**
   - Long-context inference (1M+ tokens)
   - Multi-turn conversations (persistent context)
   - Agentic AI (shared context across agents)
   - Multi-user serving (KV cache reuse across similar prompts)
8. **Key Takeaways**

- [ ] **Step 1: Web research — KV cache problem and CMX architecture**

Search for: KV cache scaling problem, CMX architecture details, NVIDIA CMX product page.
Collect: KV cache size calculations, CMX node specs, architecture descriptions.

- [ ] **Step 2: Web research — performance claims and data flow**

Search for: CMX performance benchmarks, 5x claims methodology, data flow descriptions.
Collect: benchmark data, latency numbers, throughput comparisons.

- [ ] **Step 3: Web research — alternatives and use cases**

Search for: KV cache offloading approaches, CXL for KV cache, agentic AI context requirements.
Collect: comparison data, use case descriptions, industry adoption signals.

- [ ] **Step 4: Write the full report**

Synthesize into `reports/nvidia-cmx/en/02-cmx-context-memory-platform.md`.
Target: 5000-6000 words with inline source citations.
Cross-reference: `reports/en/02-competitive-landscape.md` for SSD vendor readiness.

- [ ] **Step 5: Commit**

```bash
git add reports/nvidia-cmx/en/02-cmx-context-memory-platform.md
git commit -m "docs: add CMX Context Memory Platform deep dive report (EN)"
```

---

## Task 3: Research & Write — BlueField-4 & DOCA Memos

**Files:**
- Create: `reports/nvidia-cmx/en/03-bluefield4-doca-memos.md`

**Research queries to execute:**

- "NVIDIA BlueField-4 DPU specifications storage processor"
- "NVIDIA BlueField-4 NVMe SSD management data integrity encryption"
- "NVIDIA DOCA Memos SDK key-value API KV cache"
- "NVIDIA BlueField-4 vs BlueField-3 comparison DPU"
- "NVIDIA BlueField-4 CMX storage offload architecture"
- "NVIDIA DOCA software framework BlueField storage services"
- "DPU storage processor NVMe offload data center 2025 2026"
- "NVIDIA BlueField-4 ConnectX-8 networking capabilities"

For promising results, use WebFetch on NVIDIA developer blogs, product pages, ServeTheHome.

**Report sections to cover:**

1. **Introduction** — BlueField-4 as the brain of CMX, evolution of the DPU concept
2. **BlueField-4 Hardware Architecture**
   - Arm CPU cores (count, type, frequency)
   - ConnectX-8 networking engine (400GbE, RDMA)
   - NVMe SSD controller capabilities
   - Hardware accelerators: crypto, compression, regex, DMA engines
   - PCIe Gen6 host interface
   - Comparison to BlueField-3 (generational improvements)
3. **Storage Processor Role**
   - NVMe SSD management: namespaces, queues, wear leveling hints
   - Data integrity offload: checksums, T10-DIF/PI
   - Encryption: inline encryption/decryption for KV cache data at rest
   - Compression: reducing SSD write amplification for KV cache
   - How BlueField-4 offloads these from host CPU/GPU
4. **DOCA Memos SDK**
   - Purpose: turning Ethernet-attached flash into pod-level KV cache tier
   - Key-value APIs: put, get, delete, list operations for KV cache entries
   - Cache management: eviction policies, TTL, priority-based retention
   - Multi-tenancy: isolating KV cache across users/models/sessions
   - Integration points: TensorRT-LLM, vLLM, other inference frameworks
5. **Software Stack Architecture**
   - DOCA runtime on BlueField-4
   - Communication with GPU nodes via RDMA
   - NVMe driver and flash translation layer interaction
   - Monitoring and telemetry
6. **Key Takeaways**

- [ ] **Step 1: Web research — BlueField-4 hardware specs and evolution**

Search for: BlueField-4 specs, comparison to BF-3, hardware accelerators.
Collect: core counts, bandwidth numbers, accelerator capabilities, PCIe gen.

- [ ] **Step 2: Web research — storage processor role and DOCA Memos**

Search for: DOCA Memos SDK details, key-value APIs, storage offload architecture.
Collect: API descriptions, offload capabilities, software stack details.

- [ ] **Step 3: Web research — integration and ecosystem**

Search for: BlueField-4 ecosystem, inference framework integration, partner adoption.
Collect: integration points, partner announcements, deployment models.

- [ ] **Step 4: Write the full report**

Synthesize into `reports/nvidia-cmx/en/03-bluefield4-doca-memos.md`.
Target: 4000-5000 words with inline source citations.

- [ ] **Step 5: Commit**

```bash
git add reports/nvidia-cmx/en/03-bluefield4-doca-memos.md
git commit -m "docs: add BlueField-4 and DOCA Memos report (EN)"
```

---

## Task 4: Research & Write — Interconnects: NVLink 6 & Spectrum-X

**Files:**
- Create: `reports/nvidia-cmx/en/04-interconnects-nvlink6-spectrum-x.md`

**Research queries to execute:**

- "NVIDIA NVLink 6 specifications 130TB bandwidth switch"
- "NVIDIA NVLink 6 vs NVLink 5 comparison bandwidth latency"
- "NVIDIA Spectrum-X Ethernet RDMA RoCE AI data center"
- "NVIDIA CX9 SuperNIC specifications 800GbE"
- "NVIDIA Spectrum-X switch adaptive routing congestion control"
- "NVIDIA NVLink 6 multi-rack GPU fabric scalability"
- "NVIDIA Spectrum-X CMX RDMA KV cache access latency"
- "NVLink vs Ethernet AI data center interconnect comparison"
- "NVIDIA NetX networking rack-scale system"

For promising results, use WebFetch on NVIDIA developer blogs, Network World, NextPlatform.

**Report sections to cover:**

1. **Introduction** — Why interconnects are the backbone of disaggregated AI infrastructure
2. **NVLink 6**
   - Specifications: 130 TB/s aggregate switch bandwidth, 1800 GB/s per GPU
   - NVLink 6 Switch chip: port count, radix, power
   - Multi-rack NVLink fabric: scaling beyond single-rack with NVLink switches
   - Comparison to NVLink 5 (Blackwell): bandwidth, latency, scale improvements
   - Role in Vera Rubin pod: GPU-to-GPU communication for training and inference
3. **Spectrum-X Ethernet**
   - Architecture: Spectrum-X switch + CX9 SuperNIC
   - CX9 SuperNIC: 800GbE, RDMA offload, multi-path support
   - Spectrum-X switch: port count, bandwidth, features
   - RDMA over Converged Ethernet (RoCE): lossless fabric for storage
   - Adaptive routing: dynamic path selection for load balancing
   - Congestion control: preventing hotspots in multi-tenant environments
4. **CMX Data Path Analysis**
   - GPU → NVLink 6 → Vera CPU → PCIe Gen6 → Spectrum-X → BlueField-4 → NVMe SSD
   - Bandwidth bottlenecks at each hop
   - Latency budget: how fast can KV cache be retrieved from CMX?
   - RDMA enabling zero-copy access to remote NVMe storage
5. **NVLink vs Spectrum-X: Complementary Roles**
   - NVLink: GPU-to-GPU (training, tensor parallelism)
   - Spectrum-X: GPU-to-storage, east-west compute traffic, CMX access
   - When each is used in the inference pipeline
6. **NetX Rack-Scale Networking**
   - NetX system composition
   - How NetX connects DGX, CMX, and STX nodes
7. **Key Takeaways**

- [ ] **Step 1: Web research — NVLink 6 specifications and scaling**

Search for: NVLink 6 specs, switch details, multi-rack fabric, comparison to NVLink 5.
Collect: bandwidth numbers, latency data, port counts, scaling limits.

- [ ] **Step 2: Web research — Spectrum-X and CX9 SuperNIC**

Search for: Spectrum-X architecture, CX9 specs, RoCE, adaptive routing details.
Collect: bandwidth, port density, RDMA capabilities, congestion control mechanisms.

- [ ] **Step 3: Web research — CMX data path and NetX**

Search for: CMX data path analysis, RDMA for KV cache, NetX system details.
Collect: latency budgets, bandwidth at each hop, NetX composition.

- [ ] **Step 4: Write the full report**

Synthesize into `reports/nvidia-cmx/en/04-interconnects-nvlink6-spectrum-x.md`.
Target: 4000-5000 words with inline source citations.

- [ ] **Step 5: Commit**

```bash
git add reports/nvidia-cmx/en/04-interconnects-nvlink6-spectrum-x.md
git commit -m "docs: add NVLink 6 and Spectrum-X interconnects report (EN)"
```

---

## Task 5: Research & Write — SSD & Storage Implications

**Files:**
- Create: `reports/nvidia-cmx/en/05-ssd-storage-implications.md`

**Research queries to execute:**

- "NVIDIA CMX SSD requirements NVMe capacity endurance latency"
- "NVIDIA STX storage platform AI data center specifications"
- "KV cache SSD write pattern endurance requirements AI inference"
- "QLC SSD AI inference KV cache workload fit"
- "PCIe Gen6 SSD NVIDIA Vera Rubin CMX compatibility"
- "NVMe SSD KV cache AI inference read latency requirements"
- "NVIDIA CMX SSD vendor partner ecosystem"
- "AI inference storage requirements 2025 2026 data center"
- "SSD endurance KV cache ephemeral write workload analysis"
- "CXL SSD vs NVMe SSD AI inference comparison"

For promising results, use WebFetch on storage industry sources, NAND Research, Blocks and Files.

**Report sections to cover:**

1. **Introduction** — CMX creates a new SSD workload category: AI-native context storage
2. **CMX SSD Requirements**
   - Capacity: KV cache sizes for million-token contexts across a pod (TB-scale per node)
   - Read latency: sub-100us target for cache hit retrieval
   - Write endurance: ephemeral KV cache means high write churn but short data lifetime
   - Sequential vs random: KV cache access patterns (likely large sequential for writes, mixed for reads)
   - Interface: PCIe Gen6 NVMe, bandwidth requirements per SSD
3. **QLC SSD Fit for CMX**
   - Read-heavy retrieval aligns with QLC's strength (see [QLC Technology Fundamentals](../../en/01-qlc-technology-fundamentals.md))
   - Endurance analysis: ephemeral KV cache write patterns vs QLC P/E cycle limits
   - SLC caching relevance: burst writes to CMX match SLC cache write-back model
   - Capacity advantage: QLC's $/TB enables larger context tiers at lower cost
   - Where QLC fits vs where TLC is needed in CMX
4. **STX Platform**
   - STX (Storage eXtension) architecture and purpose
   - Difference from CMX: general-purpose AI storage vs context-specific
   - STX node composition
   - Use cases: training data staging, checkpoint storage, model repository
5. **SSD Vendor Readiness**
   - Which vendors are CMX-ready? (cross-reference [Competitive Landscape](../../en/02-competitive-landscape.md))
   - PCIe Gen6 SSD availability (cross-reference [Technology Roadmap](../../en/05-technology-roadmap-2025-2030.md))
   - Solidigm, Samsung, Micron, Kioxia/SanDisk — enterprise SSD products suited for CMX
   - Form factor considerations: E1.S, E3.S for CMX/STX nodes
6. **Impact on SSD Market**
   - New demand driver: CMX creates incremental SSD demand per AI rack
   - Estimated SSD capacity per CMX node and per pod
   - QLC SSD TAM expansion from CMX adoption
   - How CMX changes SSD procurement patterns for hyperscalers
7. **Future Directions**
   - CXL-attached memory as complementary tier (see [Technology Roadmap](../../en/05-technology-roadmap-2025-2030.md) Section 4)
   - Computational storage for KV cache processing
   - FDP/ZNS for CMX write amplification reduction
8. **Key Takeaways**

- [ ] **Step 1: Web research — CMX SSD requirements and workload analysis**

Search for: CMX SSD specs, KV cache write patterns, endurance requirements.
Collect: capacity per node, latency targets, endurance estimates, access patterns.

- [ ] **Step 2: Web research — STX platform and SSD vendor readiness**

Search for: STX details, PCIe Gen6 SSD availability, vendor CMX partnerships.
Collect: STX specs, vendor product timelines, form factor requirements.

- [ ] **Step 3: Web research — market impact and future directions**

Search for: CMX SSD demand projections, CXL for KV cache, computational storage for AI.
Collect: market sizing, CXL progress, future technology fit.

- [ ] **Step 4: Read existing QLC reports for cross-references**

Read relevant sections from:
- `reports/en/01-qlc-technology-fundamentals.md` (endurance, SLC caching sections)
- `reports/en/02-competitive-landscape.md` (vendor products section)
- `reports/en/05-technology-roadmap-2025-2030.md` (CXL, PCIe Gen6, FDP/ZNS sections)

Note specific section numbers and key data points to cross-reference.

- [ ] **Step 5: Write the full report**

Synthesize into `reports/nvidia-cmx/en/05-ssd-storage-implications.md`.
Target: 4000-6000 words with inline source citations and cross-references to QLC reports.

- [ ] **Step 6: Commit**

```bash
git add reports/nvidia-cmx/en/05-ssd-storage-implications.md
git commit -m "docs: add SSD and storage implications report (EN)"
```

---

## Task 6: Translate All Reports to Chinese

**Files:**
- Create: `reports/nvidia-cmx/zh/01-vera-rubin-pod-architecture.md`
- Create: `reports/nvidia-cmx/zh/02-cmx-context-memory-platform.md`
- Create: `reports/nvidia-cmx/zh/03-bluefield4-doca-memos.md`
- Create: `reports/nvidia-cmx/zh/04-interconnects-nvlink6-spectrum-x.md`
- Create: `reports/nvidia-cmx/zh/05-ssd-storage-implications.md`

**Translation guidelines:**
- Professional, natural Chinese — not machine-literal translation
- Technical terms: keep English acronyms with Chinese explanation on first use (e.g., "CMX（上下文内存平台）", "KV Cache（键值缓存）", "DPU（数据处理器）")
- NVIDIA product names: keep original English (BlueField-4, NVLink 6, Spectrum-X, DOCA Memos, etc.)
- Company names: keep original English
- Units and figures: keep original format
- Preserve all source citations and links (including cross-references to QLC reports)
- Preserve markdown structure and formatting

- [ ] **Step 1: Translate Report 01 — Vera Rubin Pod Architecture Overview**

Read `reports/nvidia-cmx/en/01-vera-rubin-pod-architecture.md`, translate, write to `reports/nvidia-cmx/zh/01-vera-rubin-pod-architecture.md`.

- [ ] **Step 2: Translate Report 02 — CMX Context Memory Platform**

Read `reports/nvidia-cmx/en/02-cmx-context-memory-platform.md`, translate, write to `reports/nvidia-cmx/zh/02-cmx-context-memory-platform.md`.

- [ ] **Step 3: Translate Report 03 — BlueField-4 & DOCA Memos**

Read `reports/nvidia-cmx/en/03-bluefield4-doca-memos.md`, translate, write to `reports/nvidia-cmx/zh/03-bluefield4-doca-memos.md`.

- [ ] **Step 4: Translate Report 04 — Interconnects**

Read `reports/nvidia-cmx/en/04-interconnects-nvlink6-spectrum-x.md`, translate, write to `reports/nvidia-cmx/zh/04-interconnects-nvlink6-spectrum-x.md`.

- [ ] **Step 5: Translate Report 05 — SSD & Storage Implications**

Read `reports/nvidia-cmx/en/05-ssd-storage-implications.md`, translate, write to `reports/nvidia-cmx/zh/05-ssd-storage-implications.md`.

- [ ] **Step 6: Commit**

```bash
git add reports/nvidia-cmx/zh/
git commit -m "docs: add Chinese translations for all 5 NVIDIA CMX reports"
```

---

## Task 7: Final Review & Index Update

**Files:**
- Modify: `reports/nvidia-cmx/README.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Cross-reference review**

Read all 5 English reports. Check for:
- Consistency of terminology across reports (especially NVIDIA product names)
- No contradictory claims between reports
- Cross-references between CMX reports are correct
- Cross-references to QLC SSD reports point to valid sections

- [ ] **Step 2: Update NVIDIA CMX README**

Update `reports/nvidia-cmx/README.md` with:
- Status changed to "Complete" for all reports
- One-line summary for each report
- Completion date

- [ ] **Step 3: Update CLAUDE.md**

Add NVIDIA CMX report suite to the CLAUDE.md project documentation.

- [ ] **Step 4: Final commit and push**

```bash
git add reports/nvidia-cmx/README.md CLAUDE.md
git commit -m "docs: finalize NVIDIA CMX research suite — all reports complete"
git push origin main
```
