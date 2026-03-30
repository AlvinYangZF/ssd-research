# QLC NAND Technology Fundamentals

**A comprehensive technical analysis for enterprise and data center stakeholders**

## Abstract

Quad-Level Cell (QLC) NAND flash stores four bits per cell across 16 distinct voltage states, delivering the highest bit density of any production NAND technology. This report examines QLC's cell architecture, programming mechanisms, endurance constraints, error correction strategies, SLC caching techniques, and real-world performance characteristics. As hyperscalers and enterprise buyers evaluate QLC for read-intensive and warm-storage workloads, understanding these fundamentals is essential for making informed procurement and architecture decisions. The report covers products and technology nodes current through early 2026, with specific data from Samsung, Micron, SK Hynix, Solidigm (Intel's former NAND division), SanDisk/Kioxia, and YMTC.

---

## 1. Introduction: Why QLC Matters for Storage Economics

The economics of data center storage are driven by a simple equation: cost per terabyte versus performance per terabyte. As global data creation accelerates — IDC projected 175 ZB of data by 2025 — storage architects face relentless pressure to reduce $/TB while maintaining adequate performance for an expanding range of workloads.

QLC NAND addresses this pressure directly. By storing four bits per cell (compared to three for TLC, two for MLC, and one for SLC), QLC achieves roughly 33% more bit density than TLC from the same die area. This translates to higher-capacity drives at lower cost per gigabyte. Enterprise QLC SSDs in the 15.36 TB and 30.72 TB form factors have become commercially mainstream, 61.44 TB drives entered production in 2024, and 122.88 TB drives began shipping in early 2025 ([Solidigm 122TB D5-P5336](https://news.solidigm.com/en-WW/243441-solidigm-extends-ai-portfolio-leadership-with-the-introduction-of-122tb-drive-the-world-s-highest-capacity-pcie-ssd/)). SanDisk showcased a 256 TB enterprise QLC SSD using its new UltraQLC platform at FMS 2025, with availability planned for early 2026 ([SanDisk UltraQLC at FMS 2025](https://www.sandisk.com/company/newsroom/press-releases/2025/2025-08-05-sandisk-showcases-ultraqlc-technology-platform-with-milestone-enterprise-ssd-capacity-at-fms-2025)). TrendForce reported a 400% year-on-year increase in QLC shipments for AI inference servers in 2024, reaching 30 EB ([TrendForce QLC Shipments 2024](https://www.trendforce.com/presscenter/news/20240423-12122.html)). At the drive level, QLC SSDs now approach $0.06-0.08/GB in volume enterprise pricing, though enterprise SSD contract prices surged more than 25% QoQ in Q4 2025 due to AI-driven demand ([TrendForce Q3 2025 eSSD](https://www.trendforce.com/presscenter/news/20251205-12819.html)), narrowing the gap with nearline HDDs while offering dramatically superior random read performance.

However, QLC's density advantage comes with engineering trade-offs in endurance, write performance, and error management. These trade-offs are not disqualifying — they are manageable with the right controller firmware, system architecture, and workload placement. Understanding them is what separates effective QLC deployment from disappointment.

**Business context:** QLC SSDs are not a replacement for TLC in write-intensive workloads. They are a replacement for HDDs and a complement to TLC in tiered storage architectures. The total addressable market for QLC in data centers is the vast pool of read-heavy and warm data that currently sits on spinning disk — content delivery, AI inference model storage, object stores, analytics data lakes, and archival tiers that still require sub-millisecond access.

---

## 2. Cell Architecture: Four Bits, Sixteen States

### 2.1 From SLC to QLC: The Bit-Per-Cell Progression

NAND flash memory stores data by trapping electrical charge in a floating gate or charge trap layer within each memory cell. The amount of charge determines the cell's threshold voltage (Vt), which is read by comparing it against reference voltages.

| Cell Type | Bits/Cell | Voltage States | Typical P/E Cycles | Relative Cost/GB |
|-----------|-----------|----------------|---------------------|-------------------|
| SLC       | 1         | 2              | 50,000–100,000      | 1.0x (baseline)   |
| MLC       | 2         | 4              | 3,000–10,000        | ~0.50x            |
| TLC       | 3         | 8              | 1,000–3,000         | ~0.33x            |
| QLC       | 4         | 16             | 800–1,500           | ~0.25x            |

Each additional bit per cell doubles the number of voltage states the cell must distinguish. QLC's 16 states are separated by voltage margins as narrow as 200–300 mV in mature process nodes, compared to ~500 mV for TLC and over 1 V for SLC. This narrow margin is the root cause of most QLC challenges.

### 2.2 Charge Trap Flash vs. Floating Gate

Modern 3D NAND comes in two fundamental architectures:

**Floating Gate (FG):** Used historically by Intel/Micron (now Solidigm/Micron) in their 3D NAND. Charge is stored on a conductive polysilicon floating gate surrounded by oxide. FG cells offer good charge retention and well-understood physics, but inter-cell interference (capacitive coupling between adjacent floating gates) becomes problematic as layers scale.

**Charge Trap Flash (CTF):** Used by Samsung, SK Hynix, Kioxia/WD, and YMTC. Charge is trapped in a silicon nitride layer (Si3N4) rather than a conductive gate. CTF offers several advantages for high-layer-count 3D NAND: reduced inter-cell interference (since charge is trapped in discrete locations within the nitride, not free to move across a conductor), simpler manufacturing of the cell stack, and better scalability to 200+ layers.

Starting with their 176-layer generation, Micron also transitioned to a charge trap replacement gate (CMOS-under-array with charge trap cells), converging the industry toward CTF-based architectures. As of 2024–2025, virtually all new QLC NAND production uses charge trap flash.

**Why this matters for QLC:** Charge trap cells exhibit less cell-to-cell interference than floating gate cells, which is critical when distinguishing 16 closely spaced voltage levels. However, CTF cells can suffer from a phenomenon called "charge de-trapping," where trapped electrons escape over time, gradually shifting the threshold voltage distribution. This makes data retention management more complex for QLC than for cell types with wider voltage margins.

### 2.3 3D NAND Layer Scaling

QLC's viability at scale is inseparable from 3D NAND layer count advances:

- **96-layer (2018–2019):** First generation of mass-produced QLC (Intel 660p, Samsung 870 QVO). Marginal density economics vs. TLC.
- **128-layer (2020–2021):** Improved cost structure. Micron's 128L QLC powered the Crucial P3 and early data center drives.
- **176-layer (2021–2023):** Inflection point for enterprise QLC. Micron 176L, Samsung V-NAND V7, SK Hynix 176L. Solidigm D5-P5316 (144L) reached 30.72 TB.
- **200+ layer (2023–2025):** Samsung 236-layer (V-NAND V8), Samsung 280-layer (V-NAND V9), Micron 232-layer, SK Hynix 238-layer, Kioxia/WD 218-layer BiCS8. These nodes deliver QLC die densities exceeding 1 Tb (128 GB) per die, enabling 61.44 TB and 122.88 TB form factors.
- **300+ layer (2025–2026):** SK Hynix began mass production of the world's first 321-layer QLC NAND in late 2025, featuring 2 Tb per die with 6 planes, 56% improved write speed, and 23% better power efficiency ([SK Hynix 321-Layer QLC NAND](https://news.skhynix.com/sk-hynix-begins-mass-production-of-321-layer-qlc-nand-flash/)). Samsung began building production lines for its V10 NAND at 430 layers, with commercial shipments expected in H2 2026 ([TrendForce Samsung V10](https://www.trendforce.com/news/2024/10/29/news-samsung-reportedly-plans-400-layer-vertical-nand-by-2026-targeting-1000-layer-nand-by-2030/)). Micron launched its G9 NAND (approximately 276 layers) powering the 6600 ION at 122.88 TB and a planned 245 TB variant ([Micron G9 QLC NAND](https://www.micron.com/products/storage/nand-flash/qlc-nand/g9-qlc-nand)). These advances are enabling 100+ TB SSDs and will further reduce QLC cost per bit.

Each layer count increase improves QLC economics by spreading the fixed manufacturing cost (lithography, deposition, etch) across more bits per unit of wafer area.

---

## 3. Read/Write Mechanisms

### 3.1 Programming: Incremental Step Pulse Programming (ISPP)

Writing data to a QLC cell means precisely placing its threshold voltage into one of 16 target distributions. This is accomplished through Incremental Step Pulse Programming (ISPP):

1. A programming voltage pulse (Vpgm) is applied to the selected word line.
2. A verify operation checks whether the cell's Vt has reached the target level.
3. If not, Vpgm is incremented by a small step (ΔVpgm, typically 200–400 mV) and the process repeats.
4. Cells that pass verification are inhibited from further programming.

For QLC, this process requires many more iterations than TLC because the 16 target distributions are tightly packed. A typical QLC programming operation may require 20–30+ ISPP loops, compared to 10–15 for TLC. This is a primary reason why QLC write latency is substantially higher than TLC.

### 3.2 Multi-Pass Programming

To improve programming accuracy and reduce errors, QLC NAND typically uses multi-pass (also called multi-step or coarse-fine) programming:

- **Foggy-fine programming (two-pass):** First used by Kioxia/WD. The first "foggy" pass coarsely programs cells to approximate voltage levels. The second "fine" pass precisely adjusts each cell to its target distribution. This approach reduces program disturb effects on neighboring cells.
- **Three-pass programming:** Some vendors use a three-stage approach — a coarse pass, an intermediate pass, and a final fine pass — to achieve the narrow Vt distributions QLC requires.

Multi-pass programming further increases QLC write latency. A typical QLC page program time (tProg) ranges from **1,000 to 2,000 microseconds** (1–2 ms), compared to approximately 500–700 us for TLC and 200–300 us for MLC. This 2–4x latency penalty versus TLC is the fundamental source of QLC's write performance limitations.

### 3.3 Read Operations and Reference Voltages

Reading a QLC cell requires 15 sequential sense operations (one for each boundary between the 16 voltage states), compared to 7 for TLC, 3 for MLC, and 1 for SLC. In practice, optimized read schemes reduce this:

- **Lower page reads** can often be accomplished with fewer sense operations (1–4 reads depending on the Gray-coded bit assignment).
- **Upper page reads** require the most sense operations (up to 7–8).
- Overall, a complete 4-page read of a QLC cell may require 15 individual voltage comparisons.

Typical QLC read latency (tR) is **75–120 microseconds**, compared to 50–75 us for TLC. The read penalty is much smaller than the write penalty because sense operations are fast (each takes only a few microseconds), but the cumulative effect of 15 comparisons is measurable.

### 3.4 Page Types and Data Mapping

Each QLC cell stores four bits, which are mapped to four logical pages via Gray coding:

- **Lower Page (LP)** — the least significant bit
- **Middle Page (MP)**
- **Upper Page (UP)**
- **Top Page (TP)** — the most significant bit

Gray coding ensures that adjacent voltage states differ by only one bit, minimizing the number of bit errors when a cell's voltage drifts slightly. The choice of Gray code mapping also affects which pages are faster to read and more reliable, influencing firmware-level data placement strategies.

---

## 4. Endurance Challenges

### 4.1 Program/Erase (P/E) Cycle Limits

QLC NAND is rated for approximately **800 to 1,500 P/E cycles** depending on the specific NAND generation, vendor, and process node. Some enterprise-optimized QLC parts (e.g., Solidigm D5-P5336) claim up to 1,500 cycles with advanced controller algorithms. For context:

- SLC: 50,000–100,000 P/E cycles
- MLC: 3,000–10,000 P/E cycles
- TLC: 1,000–3,000 P/E cycles
- QLC: 800–1,500 P/E cycles

These numbers represent the point at which the raw bit error rate (RBER) exceeds the capability of the drive's error correction to maintain data integrity within acceptable limits (typically an uncorrectable bit error rate, or UBER, of less than 10^-17 for enterprise drives).

### 4.2 Oxide Degradation Mechanisms

Each P/E cycle damages the tunnel oxide layer through which electrons pass during programming and erasure:

**Trap generation:** High-energy electrons (hot carriers) during Fowler-Nordheim tunneling create defect sites (traps) in the SiO2 tunnel oxide and at the Si/SiO2 interface. These traps:
- Shift the cell's threshold voltage over time (Vt drift)
- Increase variability in the Vt distribution (wider distributions)
- Create charge leakage paths that reduce data retention

**Charge de-trapping (CTF-specific):** In charge trap flash cells, the stored charge can gradually de-trap from the silicon nitride layer, causing the Vt to shift downward over time. The rate of de-trapping depends on temperature, time, and the number of prior P/E cycles.

**Interface state generation:** Repeated cycling generates interface states at the substrate-oxide boundary, which alter the cell's I-V characteristics and increase read noise.

For QLC, these mechanisms are especially impactful because the 16 Vt distributions have minimal margin. Even modest oxide degradation causes the distributions to widen and overlap, dramatically increasing the bit error rate. A QLC cell at 1,000 P/E cycles may exhibit a raw BER 10–100x higher than the same cell when fresh.

### 4.3 Data Retention and Temperature Effects

QLC data retention — the time data remains readable without power — is strongly affected by P/E cycle count and storage temperature. Enterprise specifications typically require:

- **Fresh cells (< 100 P/E cycles):** 1 year at 40C (unpowered)
- **End-of-life cells (at rated P/E limit):** 3 months at 40C (unpowered)

At elevated temperatures (55–85C, common in data center environments), charge loss accelerates, reducing retention time. Enterprise QLC SSDs compensate with:

- **Background patrol reads:** Periodically scanning data and refreshing (rewriting) blocks approaching error thresholds
- **Adaptive read voltage tracking:** Adjusting reference voltages to track Vt distribution drift
- **Read reclaim / data refresh:** Proactively moving data from blocks with increasing error rates to fresher blocks

### 4.4 Drive Write Per Day (DWPD) Implications

The low P/E cycle count translates directly to limited write endurance at the drive level. The Drive Writes Per Day (DWPD) metric expresses how many times the drive's entire capacity can be written per day over its warranty period (typically 5 years). For enterprise QLC SSDs:

- **Typical QLC:** 0.3–1 DWPD (e.g., Solidigm D5-P5316: 0.58 DWPD; Solidigm D5-P5336: 0.58 DWPD at 5-year warranty)
- **Typical TLC enterprise:** 1–3 DWPD
- **High-endurance TLC:** 3–10 DWPD

A 15.36 TB QLC drive at 0.5 DWPD can sustain approximately 7.68 TB of writes per day for five years — roughly 14 PB total bytes written (TBW). This is more than adequate for read-heavy workloads (CDN caches, AI model serving, media streaming, warm object storage) where the write ratio is well below 0.5 DWPD.

---

## 5. Error Correction: LDPC and Beyond

### 5.1 Why QLC Demands Advanced ECC

QLC's narrow voltage margins and high raw bit error rates (RBER) necessitate significantly stronger error correction than prior NAND generations. Typical RBER values:

- **SLC:** 10^-6 to 10^-5 (1 error per million bits)
- **MLC:** 10^-5 to 10^-4
- **TLC:** 10^-4 to 10^-3
- **QLC (fresh):** ~10^-3
- **QLC (end of life):** 10^-2 to 10^-1 (1–10 errors per 100 bits)

The enterprise requirement is UBER < 10^-17 (less than 1 unrecoverable error per 100 petabits read). Bridging the gap from a RBER of 10^-2 to a UBER of 10^-17 demands ECC capable of correcting a very large number of errors per codeword.

### 5.2 LDPC (Low-Density Parity-Check) Codes

Modern QLC SSD controllers universally use LDPC codes, which offer near-Shannon-limit error correction performance. Key aspects:

**Hard-decision vs. soft-decision decoding:**

- **Hard-decision decoding** reads each cell as a single voltage value and determines a hard 0/1 for each bit. This is fast (single read) but limited in correction capability.
- **Soft-decision decoding** performs multiple reads at slightly shifted reference voltages (typically 3–7 reads per sense operation) to obtain reliability information (log-likelihood ratios) for each bit. This soft information enables the LDPC decoder to correct significantly more errors — typically 2–3x the correction capability of hard-decision decoding — at the cost of increased read latency and controller processing.

QLC SSDs rely heavily on soft-decision decoding, especially as the NAND ages. A typical read flow:

1. **First attempt:** Hard-decision decode. If successful (low error count), return data immediately. Latency: ~100 us.
2. **Second attempt (if hard fails):** Soft-decision decode with 3 additional reads. Correction capability increases substantially. Latency: ~300–500 us.
3. **Third attempt (if soft fails):** Extended soft-decision decode with 7+ reads and maximum-iteration LDPC decoding. Latency: ~1–2 ms.
4. **Last resort:** RAID-like reconstruction from internal parity (if available) or host-level redundancy.

### 5.3 Code Rate and Overhead

LDPC codes for QLC use lower code rates (higher redundancy) than those for TLC:

- **TLC:** Typically ~0.9 code rate (10% parity overhead). For a 16 KB data page, approximately 1.6 KB of ECC parity is stored.
- **QLC:** Typically ~0.85–0.88 code rate (12–15% parity overhead). This increased parity consumes part of the raw capacity advantage QLC provides.

The over-provisioning (OP) for enterprise QLC drives is also typically higher than for TLC drives — often 10–28% of raw capacity is reserved for ECC, metadata, SLC cache, and spare blocks, compared to 7–15% for enterprise TLC.

### 5.4 Internal RAID / XOR Redundancy

Many enterprise QLC SSDs implement RAID-like parity protection within the drive:

- **Solidigm's "RAIN" (Redundant Array of Independent NAND):** Stripes data across multiple NAND dies with XOR parity, enabling recovery from a complete die failure.
- **Samsung's internal parity scheme:** Similar per-die parity protection.
- **Micron's FlexRAID:** Parity at the die and plane level.

This internal redundancy is especially important for QLC because the higher error rates mean individual page or block failures are more frequent. Internal RAID provides a safety net below the host-level data protection.

---

## 6. SLC Caching: Bridging the Write Performance Gap

### 6.1 The Fundamental Problem

Raw QLC write performance — with tProg of 1–2 ms and multi-pass programming — cannot match the sustained write bandwidth that enterprise applications expect. Direct-to-QLC write speeds typically plateau at **100–400 MB/s** for a single drive, well below what the NVMe interface can deliver. SLC caching solves this by initially writing incoming data in SLC mode (1 bit per cell) and later migrating it to QLC.

**Emerging alternative — Direct QLC Write:** SanDisk's UltraQLC platform, announced at FMS 2025, introduces "Direct Write QLC" technology that bypasses the pseudo-SLC buffer entirely, performing power-loss-safe writes directly to QLC on the first pass. This approach eliminates folding overhead and the associated write amplification, though it requires advanced controller and firmware co-optimization ([SanDisk UltraQLC Blog](https://www.sandisk.com/company/newsroom/blogs/2025/inside-ultraqlc-the-enterprise-ssd-platform-engineered-for-ai)).

### 6.2 Static vs. Dynamic SLC Cache

**Static SLC cache:** A fixed portion of the NAND is permanently configured as SLC. For example, on a 15.36 TB QLC drive, 200–500 GB might be permanently allocated as SLC cache. Advantages: predictable performance; the cache is always available. Disadvantages: reduces usable QLC capacity; cache size is fixed regardless of workload.

**Dynamic SLC cache:** The controller dynamically designates unused (empty) QLC blocks as temporary SLC blocks. When the drive is mostly empty, a very large proportion of writes can be absorbed in SLC mode. As the drive fills, the dynamic SLC cache shrinks. Most modern enterprise QLC SSDs use dynamic SLC caching, sometimes combined with a small static reserve.

**Typical cache sizes:**
- Consumer QLC SSDs: 12–72 GB dynamic SLC cache (varies with drive capacity and fill level)
- Enterprise QLC SSDs: 100–500+ GB, often dynamically scaled

### 6.3 Folding: SLC-to-QLC Data Migration

"Folding" is the process of reading data from SLC-mode cells and reprogramming it into QLC-mode cells. This is a background operation performed by the controller when:

- The SLC cache is approaching capacity
- The drive is idle or under low load
- The controller's garbage collection needs to reclaim SLC blocks

Folding is expensive:
1. Read the SLC data (fast, ~25 us per page).
2. Assemble 4 pages worth of data (since QLC packs 4 bits per cell, 4 SLC pages fold into 1 QLC word-line's 4 pages).
3. Program the QLC cells using multi-pass ISPP (slow, 1–2 ms per word line).
4. Verify the QLC programming succeeded.
5. Erase the SLC source blocks (erase time: 3–10 ms per block).

The folding process consumes NAND bandwidth and introduces write amplification (data is written once in SLC, then again in QLC — a minimum write amplification factor of 2x for cached writes, before accounting for garbage collection).

### 6.4 Performance Cliffs

When the SLC cache is exhausted — because sustained writes exceeded the cache size and the controller cannot fold fast enough — the drive must write directly to QLC. This creates a dramatic performance cliff:

- **Burst performance (in SLC cache):** 3,000–5,000 MB/s sequential write (NVMe Gen4), comparable to TLC drives
- **Sustained performance (direct-to-QLC):** 100–500 MB/s sequential write, depending on the controller and NAND generation

This 5–30x performance drop is the most visible QLC characteristic in benchmarks and real-world testing. Enterprise QLC drives (e.g., Solidigm D5-P5316, Samsung PM9D3, Micron 6500 ION) mitigate this with:

- **Larger SLC caches** (often 1–3% of total capacity, which at 30.72 TB is 300–900 GB)
- **Faster controllers** that can fold more aggressively during mixed workloads
- **Workload-aware firmware** that prioritizes folding during detected idle periods
- **Write bandwidth limiting** that throttles host writes to prevent cache exhaustion

For properly matched workloads (read-heavy, with write bursts that fit within the SLC cache), the performance cliff is rarely encountered. This is why workload characterization is essential before QLC deployment.

---

## 7. Wear Leveling, Read Disturb, and Data Retention

### 7.1 Wear Leveling

Wear leveling distributes P/E cycles evenly across all NAND blocks to prevent any single block from wearing out prematurely. This is critical for QLC due to the limited P/E budget:

**Dynamic wear leveling:** Only distributes writes among free blocks. Hot data (frequently overwritten) cycles through many blocks. Effective for write-heavy regions but does not address cold data sitting in blocks that are never erased.

**Static (global) wear leveling:** Periodically moves cold data from low-cycle blocks to high-cycle blocks, freeing the low-cycle blocks for new writes. This is essential for QLC because even a small number of "hot" blocks can exhaust their P/E budget if not actively managed.

Enterprise QLC controllers implement sophisticated wear leveling algorithms that track per-block P/E counts and proactively rebalance data to keep all blocks within a narrow cycle-count range. The target is to reach the drive's rated TBW with all blocks approximately equally worn.

### 7.2 Read Disturb

Read disturb occurs when reading one word line (a row of cells) in a NAND block causes a small Vt shift in cells on neighboring, unread word lines. This happens because the read operation applies a pass voltage (Vpass, typically 6–8V) to the unselected word lines, which is high enough to cause slow, cumulative charge injection.

For QLC, read disturb is a heightened concern because:
- The narrow voltage margins mean even small Vt shifts can cause bit errors
- Data center workloads can involve billions of reads to the same data (e.g., popular content in a CDN cache)
- QLC blocks contain more pages per block (typically 4x as many logical pages as SLC mode), increasing the number of read operations each block experiences

**Mitigation strategies:**
- **Read count tracking:** The controller counts reads per block. After a threshold (typically 50,000–200,000 reads), the block is scheduled for read reclaim.
- **Read reclaim (data refresh):** Data from a high-read-count block is read, error-corrected, and rewritten to a fresh block. The original block is erased.
- **Adaptive Vpass optimization:** Some controllers dynamically adjust the pass voltage to minimize disturb while maintaining reliable reads.

Read reclaim consumes P/E cycles, creating a trade-off: preventing read disturb errors costs endurance. For extremely read-heavy QLC workloads, the P/E cycles consumed by read reclaim must be budgeted alongside host writes.

### 7.3 Garbage Collection Impact

Garbage collection (GC) consolidates valid data from partially-used blocks into new blocks, then erases the old blocks. GC-induced write amplification (WAF) is particularly impactful for QLC:

- QLC blocks are large (typically 48–96 MB per block in current 200+ layer NAND, compared to 12–24 MB for TLC due to QLC having 4x the pages per word line).
- Larger blocks mean more valid data must be relocated during GC, increasing write amplification.
- Higher WAF consumes more P/E cycles from an already limited budget.

Enterprise QLC SSDs typically target WAF of 1.5–3x for typical enterprise workloads, achieved through:
- Over-provisioning (more spare blocks reduces the frequency and intensity of GC)
- Write coalescing in the SLC cache (batching small writes before folding to QLC)
- Host-assisted placement (Zoned Namespaces / ZNS, or Flexible Data Placement / FDP) that reduces GC by aligning data lifetimes with erase block boundaries

ZNS and FDP are emerging NVMe standards that allow the host to manage data placement, potentially reducing QLC write amplification to near 1.0x. This could be transformative for QLC endurance — effectively doubling or tripling usable write life.

---

## 8. Performance Characteristics

### 8.1 Sequential Performance

Enterprise QLC SSDs deliver excellent sequential read performance that matches or approaches TLC drives:

| Metric | Enterprise QLC (PCIe Gen4) | Enterprise TLC (PCIe Gen4) |
|--------|---------------------------|---------------------------|
| Sequential Read | 6,000–7,000 MB/s | 6,500–7,000 MB/s |
| Sequential Write (SLC cache) | 2,000–5,000 MB/s | 3,000–5,500 MB/s |
| Sequential Write (direct QLC) | 100–500 MB/s | N/A (TLC direct: 1,500–2,500 MB/s) |

Representative products (2024–2025):
- **Solidigm D5-P5336 (122.88 TB, PCIe Gen4):** 7,000 MB/s read, 3,300 MB/s write (SLC cache). Priced starting at ~$12,400 (~$0.10/GB) ([TechRadar Solidigm 122TB](https://www.techradar.com/pro/worlds-largest-ssd-is-on-sale-for-almost-usd12-400-and-yes-it-is-quite-a-bargain-if-you-can-afford-it-of-course)).
- **Micron 6600 ION (122.88 TB, PCIe Gen5):** Built on G9 QLC NAND with 245 TB variant planned for H1 2026 ([Micron G9 NAND Blog](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)).
- **Samsung PM9D3 (15.36 TB):** 6,000 MB/s read, 2,000 MB/s write
- **SanDisk UltraQLC (256 TB, U.2):** Announced at FMS 2025 using BiCS8 QLC CBA NAND with direct QLC writes; availability early 2026 ([SanDisk FMS 2025](https://investor.sandisk.com/news-releases/news-release-details/sandisk-showcases-ultraqlctm-technology-platform-milestone)).

### 8.2 Random Performance

Random I/O performance is where QLC and TLC diverge more significantly, particularly for writes:

| Metric | Enterprise QLC | Enterprise TLC |
|--------|---------------|---------------|
| Random Read (4K, QD128) | 900K–1,200K IOPS | 1,000K–1,600K IOPS |
| Random Write (4K, QD128, SLC cache) | 100K–250K IOPS | 200K–400K IOPS |
| Random Mixed (70/30 R/W) | 200K–400K IOPS | 350K–600K IOPS |

QLC random read performance at high queue depths is within 70–90% of TLC, making QLC well-suited for random-read workloads like database index scans, key-value store reads, and content serving.

### 8.3 Latency Profiles

Latency is where QLC's underlying physics are most apparent:

| Operation | QLC Typical | TLC Typical | SLC Typical |
|-----------|-------------|-------------|-------------|
| Read (tR) | 75–120 us | 50–75 us | 25–30 us |
| Program (tProg) | 1,000–2,000 us | 500–700 us | 200–300 us |
| Erase (tErase) | 10–15 ms | 3–5 ms | 1–3 ms |

**Read tail latency** is a critical concern for enterprise workloads. When a QLC read requires soft-decision decoding (more likely as the NAND ages), read latency jumps from ~100 us to 300–500 us or even 1+ ms. At the P99.9 and P99.99 levels, QLC drives can exhibit tail latencies 3–10x higher than their median read latency.

Enterprise QLC controllers manage tail latency through:
- **Read retry optimization:** Caching optimal read voltage offsets per block to minimize the need for soft-decision decoding
- **Adaptive voltage tracking:** Continuously updating reference voltages to track Vt drift, keeping more reads within hard-decision correction range
- **Priority arbitration:** Allowing urgent host reads to preempt background operations (GC, folding)

### 8.4 Read/Write Asymmetry

QLC exhibits the most extreme read/write performance asymmetry of any NAND type:

- **Read bandwidth** can exceed write bandwidth by 5–15x when writing directly to QLC
- **Read IOPS** can exceed write IOPS by 4–10x
- **Read latency** is 10–20x lower than write latency at the NAND level

This asymmetry is not a defect — it is a design characteristic that makes QLC ideal for read-dominant workloads. The most successful enterprise QLC deployments explicitly match this asymmetry to workload requirements:

- **CDN and media serving:** 95%+ reads. QLC excels.
- **AI/ML model serving:** Model weights are written once and read millions of times. QLC excels.
- **Data lake / analytics:** Write-once-read-many pattern. QLC is cost-effective.
- **OLTP databases:** 50/50 or higher write ratios. QLC is a poor fit; use TLC or MLC.

---

## 9. Industry Innovations (2024–2025)

### 9.1 Solidigm (Intel NAND Heritage)

Solidigm (owned by SK Hynix since 2021, operating Intel's former NAND business) has been the most aggressive proponent of enterprise QLC:

- **D5-P5336 (122.88 TB):** Solidigm extended its flagship QLC SSD to 122.88 TB in late 2024 / early 2025 — the world's highest-capacity PCIe SSD at launch — based on 192-layer QLC NAND with NVMe 2.0 and PCIe Gen4 x4. The drive offers "unlimited write durability" (no DWPD cap for typical read-intensive workloads). Solidigm's roadmap targets 245 TB-class SSDs by end of 2026 ([TechRadar Solidigm 245TB roadmap](https://www.techradar.com/pro/solidigm-confirms-245-tb-ssds-set-to-launch-before-end-of-2026)).
- **Synth workload optimization:** Solidigm's firmware includes workload-aware algorithms that detect access patterns and optimize SLC cache management, read voltage tracking, and GC scheduling accordingly.
- **PLC demonstration:** Solidigm demonstrated the world's first PLC (5-bit-per-cell) SSD prototype at Flash Memory Summit 2022, signaling long-term intent beyond QLC ([Solidigm PLC Demo](https://news.solidigm.com/en-WW/217006-solidigm-demonstrates-world-s-first-penta-level-cell-ssd-at-flash-memory-summit/)).

### 9.2 Samsung

- **V-NAND V9 (280-layer QLC):** Samsung began mass production of its 9th-gen V-NAND with 280-layer QLC in September 2024, achieving approximately 86% higher bit density than the prior QLC generation, doubled write performance, and 30–50% reduction in read/write power consumption ([Samsung V9 QLC Press Release](https://news.samsung.com/global/samsung-begins-industrys-first-mass-production-of-qlc-9th-gen-v-nand-for-ai-era)). However, TrendForce reported that full-scale V9 QLC commercialization was delayed to at least H1 2026 due to design challenges ([TrendForce Samsung V9 Delay](https://www.trendforce.com/news/2025/09/16/news-samsung-reportedly-delays-v9-qlc-nand-to-1h26-as-sk-hynix-hits-321-layers-in-august/)).
- **V10 (430-layer) roadmap:** Samsung is building a V10 production line targeting 430 layers with hybrid bonding and ultra-low-temperature etching, with commercial shipments expected in H2 2026 ([Tom's Hardware Samsung V10](https://www.tomshardware.com/pc-components/ssds/samsung-plans-big-capacity-jump-for-ssds-preps-290-layer-v-nand-this-year-430-layer-for-2025)). Samsung has also disclosed ambitions for 1,000-layer NAND by 2030 ([TrendForce Samsung 1000 Layers](https://www.trendforce.com/news/2025/02/26/news-samsung-reportedly-targets-1000-layer-nand-by-2030-rolls-out-wafer-bonding-at-400-layers/)).
- **PM9D3:** Samsung's enterprise QLC SSD in U.2 and E3.S form factors, targeting read-intensive data center workloads up to 15.36 TB.
- **BM1743:** Samsung's enterprise QLC offering optimized for hyperscaler deployments, supporting PCIe Gen5 interface.

### 9.3 Micron

- **G9 QLC NAND:** Micron launched its 9th-generation (G9) QLC NAND in 2025, powering a new family of data center SSDs. The G9 node uses Micron's CMOS-under-array architecture for industry-leading die density ([Micron G9 QLC NAND](https://www.micron.com/products/storage/nand-flash/qlc-nand/g9-qlc-nand)).
- **6600 ION:** The successor to the 6500 ION, built on G9 QLC NAND with PCIe Gen5 interface. Available in 30.72 TB, 61.44 TB, and 122.88 TB (E3.S), with a 245 TB variant (E3.L) planned for H1 2026. In early 2026, the 122 TB and 245 TB models were entering qualification at multiple hyperscale customers ([Micron G9 Blog](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)).
- **9650 (PCIe Gen6):** Micron launched the world's first PCIe Gen6 data center SSD in February 2026, delivering 28 GB/s sequential reads — double the Gen5 generation. Available in liquid-cooled E1.S variants for AI servers ([Micron 9650 Gen6](https://www.storagereview.com/news/micron-9650-nvme-ssd-enters-mass-production-as-first-pcie-gen6-enterprise-ssd)).
- **CXL-enabled QLC exploration:** Micron has demonstrated CXL-attached QLC NAND as a memory tier, pointing toward future architectures where QLC serves as a persistent memory/storage tier below DRAM.

### 9.4 SK Hynix

- **321-layer 4D NAND QLC:** SK Hynix began mass production of the world's first 300+ layer QLC NAND in late 2025, achieving 2 Tb per die with 6 planes (up from 4). Write speed improved 56%, read speed 18%, and write power efficiency improved 23% ([SK Hynix 321-Layer QLC](https://news.skhynix.com/sk-hynix-begins-mass-production-of-321-layer-qlc-nand-flash/)). The company plans to apply 321-layer NAND first to PC SSDs, then enterprise SSDs and smartphones.
- **AI NAND strategy:** SK Hynix unveiled its "AIN" (AI NAND) strategy, including an "AIN D" (Density) solution using 3D QLC NAND designed to reach petabyte-level drive capacities. The company is developing a 244 TB enterprise SSD product using 321-layer QLC NAND ([Tom's Hardware SK Hynix AI NAND](https://www.tomshardware.com/pc-components/ssds/sk-hynix-unveils-ai-nand-strategy-including-gargantuan-petabyte-class-qlc-ssds-ultra-fast-hbf-and-100m-iops-ssds-also-in-the-pipeline)).
- **PE8000 series:** Enterprise QLC SSDs targeting hyperscaler read-intensive workloads.

### 9.5 SanDisk (formerly WD Flash) / Kioxia

- **BiCS8 (218-layer):** Kioxia and SanDisk's (formerly Western Digital's flash business, separated into an independent company in early 2025) latest generation. QLC mode enables high-density enterprise and consumer drives.
- **UltraQLC platform:** SanDisk announced its UltraQLC technology platform at FMS 2025, featuring the SN670 (128 TB) and a 256 TB enterprise SSD in U.2 form factor. Key innovations include Direct Write QLC (bypassing SLC cache), dynamic frequency scaling (~10% performance gain), and optimized data-retention profiles reducing refresh cycles by ~33% ([SanDisk UltraQLC](https://investor.sandisk.com/news-releases/news-release-details/sandisk-showcases-ultraqlctm-technology-platform-milestone)). Availability is planned for H1 2026.
- **XD8 series:** Enterprise QLC SSDs in E3.S form factor for data center deployment.

### 9.6 YMTC (China)

- **X3-9070 (232-layer):** YMTC's latest generation, primarily TLC but with QLC capability. Supply is limited to the China domestic market due to US export controls on semiconductor equipment (October 2022 restrictions). YMTC's QLC is primarily used in consumer SSDs within China; enterprise QLC adoption is still nascent.

---

## 10. Key Takeaways

### For Technical Engineers

1. **QLC stores 4 bits/cell across 16 voltage states** with margins as narrow as 200–300 mV. This tight spacing drives most of QLC's engineering challenges — endurance, error rates, and write latency.

2. **P/E endurance of 800–1,500 cycles** is sufficient for read-heavy workloads (0.3–1 DWPD). Calculate your actual workload's write volume carefully; many "mixed" workloads still write far less than 0.5 DWPD.

3. **LDPC with soft-decision decoding** is mandatory for QLC. Plan for tail latency increases at high P/E counts when soft-decision reads become more frequent. Monitor P99.9 read latency, not just averages.

4. **SLC cache sizing and folding behavior** determine burst write performance. Test with sustained writes that exceed the cache to understand worst-case behavior. Direct-to-QLC write speed (100–500 MB/s) is the true floor.

5. **ZNS and FDP** can dramatically reduce write amplification for QLC, potentially extending effective endurance by 2–3x. Evaluate these interfaces for new deployments.

6. **Read disturb is a real concern** for hot read data. Ensure the drive's firmware includes read reclaim, and budget the associated P/E cycle overhead.

### For Business and Product Leaders

1. **QLC reduces storage cost per TB by 20–40% versus TLC** at the drive level, and more at the system level due to higher capacity per drive slot (fewer drives, controllers, cables, and rack units needed for the same capacity).

2. **QLC replaces HDDs, not TLC.** Position QLC for the 70–80% of data center data that is read-heavy or warm: CDN caches, AI model repositories, object stores, media libraries, and analytics datasets.

3. **Total cost of ownership (TCO) favors QLC** when power, cooling, and density are factored in. A 61.44 TB QLC SSD in an E1.L slot replaces 6–8 nearline HDDs with lower power consumption, higher rack density, and faster access times.

4. **Endurance is manageable but must be planned.** A 30.72 TB QLC drive at 0.5 DWPD writes ~15 TB/day for 5 years. If your workload writes less than that per drive, QLC endurance is not a constraint.

5. **Performance cliffs are avoidable** with proper workload placement. Do not deploy QLC for write-intensive workloads. Use tiered architectures: TLC/Optane for write-heavy tiers, QLC for read-heavy and capacity tiers.

6. **The 200+ layer generation (2024–2025) is the enterprise QLC inflection point.** Capacity, cost, and reliability have matured sufficiently for mainstream data center deployment. Companies that delay QLC adoption risk overpaying for storage relative to competitors who adopt it strategically.

---

## Sources and References

- JEDEC JESD218 — Solid State Drive Requirements and Endurance Test Method
- JEDEC JESD219 — Solid State Drive Endurance Workloads
- Solidigm D5-P5336 product brief and specifications ([solidigm.com](https://www.solidigm.com/products/data-center.html))
- Micron 6500 ION SSD product specifications ([micron.com](https://www.micron.com/products/storage/ssd/data-center-ssd))
- Samsung PM9D3 / BM1743 product briefs ([semiconductor.samsung.com](https://semiconductor.samsung.com/ssd/enterprise-ssd/))
- Micheloni, R., Crippa, L., & Marelli, A. — *Inside NAND Flash Memories*, Springer
- Cai, Y., et al. — "Error Characterization, Mitigation, and Recovery in Flash-Memory-Based Solid-State Drives," Proceedings of the IEEE, 2017
- Cai, Y., et al. — "Vulnerabilities in MLC NAND Flash Memory Programming: Experimental Analysis, Exploits, and Mitigation Techniques," HPCA, 2017
- Zuolo, L., et al. — "Solid-State Drive Management and Reliability" — survey of SSD reliability mechanisms
- AnandTech — extensive QLC SSD review coverage ([anandtech.com](https://www.anandtech.com))
- StorageReview — enterprise SSD benchmarks and analysis ([storagereview.com](https://www.storagereview.com))
- Flash Memory Summit proceedings (2023, 2024) — presentations from Samsung, Micron, SK Hynix, Solidigm on QLC roadmaps

---

*Report prepared and updated March 2026. Technology specifications reflect products announced or shipping through early 2026. Future roadmap items (400+ layer NAND, PLC) are based on vendor disclosures and analyst projections, and are subject to change.*
