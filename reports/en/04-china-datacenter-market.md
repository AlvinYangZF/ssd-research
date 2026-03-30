# China Data Center Market for QLC SSDs

**Abstract:** China operates the world's second-largest data center market by capacity and is on a trajectory to narrow the gap with the United States through the end of the decade. This report analyzes the unique dynamics shaping QLC SSD adoption within China's data center ecosystem — from the domestic NAND supply chain anchored by YMTC and constrained by US export controls, to the procurement strategies of hyperscale cloud operators (Alibaba, Tencent, Baidu, Huawei, ByteDance), government localization mandates under the "dual circulation" framework, and a growing ecosystem of domestic SSD vendors. The interplay between geopolitical risk, technology ambition, and market scale makes China's data center SSD market distinct from any other geography. For QLC SSD vendors — domestic and foreign — China represents both the highest-growth opportunity and the most complex operating environment through 2030.

---

## 1. Market Overview

### 1.1 Market Size and Growth Trajectory

China's data center SSD market has been one of the fastest-growing segments in the global storage industry. Industry estimates placed the broader China enterprise SSD market at approximately $8–10 billion in 2024, with data center SSDs (cloud, hyperscale, colocation) representing roughly 60–65% of that figure. By 2030, the total addressable market for data center SSDs in China is projected to reach $18–22 billion, driven by cloud expansion, AI infrastructure buildout, and the ongoing HDD-to-SSD transition in warm and cold storage tiers.

Several structural factors distinguish the Chinese market:

- **Scale of buildout.** China added an estimated 3–4 million new server racks between 2022 and 2025. The "Eastern Data, Western Computing" (东数西算) national initiative, launched in February 2022, designated eight national computing hubs and ten data center clusters, catalyzing a construction wave that is still accelerating as of 2025.
- **AI-driven demand surge.** The proliferation of large language models from Baidu (ERNIE), Alibaba (Tongyi Qianwen), ByteDance (Doubao), and others has sharply increased storage requirements for training datasets, model checkpoints, and inference caching. AI-related storage demand in Chinese data centers is estimated to grow at a 35–45% CAGR through 2028.
- **Localization pressure.** Government procurement preferences and supply chain security concerns increasingly favor domestically produced SSDs using Chinese NAND flash, creating a bifurcated market where foreign and domestic products coexist under different competitive dynamics.
- **QLC-specific opportunity.** QLC SSDs are particularly well-suited to China's growth pattern: the largest volume demand comes from read-heavy cloud storage, content delivery, surveillance data archival, and AI training data lakes — all workloads where QLC's cost-per-terabyte advantage over TLC is compelling, and write endurance requirements are modest.

### 1.2 Unique Market Dynamics

The Chinese data center SSD market operates under constraints and incentives not present in Western markets. Procurement decisions are shaped not only by price, performance, and reliability — the standard vendor evaluation criteria — but also by regulatory compliance with data localization laws (the Cybersecurity Law of 2017, the Data Security Law of 2021, and the Personal Information Protection Law of 2021), government incentives for domestic component adoption, and strategic hedging against potential supply disruptions caused by export controls.

This creates a tiered market structure:

1. **Tier 1 — Government and state-owned enterprises (SOEs):** Strong mandate for domestic SSDs. YMTC NAND + domestic controllers preferred or required.
2. **Tier 2 — Major cloud providers (Alibaba, Tencent, Baidu, Huawei):** Pragmatic mix of imported and domestic SSDs, shifting toward domestic over time.
3. **Tier 3 — Private enterprises and smaller cloud providers:** Price-sensitive, historically reliant on Samsung, Micron, and SK Hynix SSDs, but increasingly adopting domestic alternatives as quality improves and pricing becomes competitive.

---

## 2. Domestic NAND Ecosystem

### 2.1 YMTC — The Cornerstone

Yangtze Memory Technologies Co., Ltd. (YMTC), headquartered in Wuhan, is China's sole NAND flash manufacturer at scale. Founded in 2016 with approximately $24 billion in cumulative state investment (through the National Integrated Circuit Industry Investment Fund, known as the "Big Fund," and Hubei provincial funding), YMTC has made remarkable progress in closing the technology gap with established players.

**Xtacking Architecture.** YMTC's signature technical innovation is Xtacking, a hybrid bonding architecture that fabricates the NAND cell array and the peripheral logic circuits (I/O, page buffers, charge pumps) on separate wafers, then bonds them together using through-silicon vias (TSVs) at extremely high density. This approach offers two key advantages: (1) each wafer can be independently optimized for its respective process node — the NAND array on a specialized high-aspect-ratio etching process, and the periphery on a more advanced logic node for faster I/O; (2) the resulting die is more compact, yielding better storage density per unit area. Xtacking 3.0, announced in late 2023, further improved bonding density and enabled YMTC to integrate more sophisticated ECC and signal processing logic directly beneath the array.

**Layer Count Progress.** YMTC's production timeline:
- **64-layer (Xtacking 1.0):** Volume production began in 2019–2020. This generation powered YMTC's initial consumer SSD products and established manufacturing confidence.
- **128-layer (Xtacking 2.0):** Ramped to volume production in 2021–2022. Competitive with Samsung's 128-layer V-NAND and Micron's 176-layer products in density terms, partly due to the Xtacking die-stacking advantage.
- **232-layer (Xtacking 3.0):** YMTC demonstrated 232-layer 3D NAND samples in 2023. Volume production has been constrained by US export controls (discussed in Section 3), though YMTC has made significant progress. As of 2025, YMTC has advanced to 270-layer NAND and reportedly accelerated production of 294-layer devices, rapidly narrowing the technological gap with Korean competitors ([Digitimes YMTC Equipment Localization](https://www.digitimes.com/news/a20251231PD214/ymtc-3d-nand-equipment-production-localization.html)). YMTC's total NAND output reached approximately 130,000–150,000 wafer starts per month by 2025 (~8% of global NAND supply), with the company targeting 15% of global NAND supply by late 2026 ([Tom's Hardware YMTC 15% Target](https://www.tomshardware.com/pc-components/ssds/chinas-ymtc-moves-to-break-free-of-u-s-sanctions-by-building-production-line-with-homegrown-tools-aims-to-capture-15-percent-of-nand-market-by-late-2026)). However, YMTC's market share fell below 5% in Q2 2025 due to ongoing production capacity constraints ([Tom's Hardware YMTC Struggles](https://www.tomshardware.com/pc-components/ssds/chinas-premier-memory-maker-ymtc-struggles-amid-chokehold-of-us-sanctions-outdated-chipmaking-tools-and-lack-of-new-tools-has-hampered-production-and-development-of-new-tech)).

**QLC Progress.** YMTC has developed QLC NAND on its 128-layer platform, with engineering samples circulating to domestic SSD partners since at least 2023. The 128-layer QLC die supports 1 Tbit density per die, enabling 4 TB and 8 TB SSD configurations in standard form factors. However, YMTC's QLC has been primarily adopted in consumer products and select enterprise read-caching applications. For the data center market, YMTC's QLC faces two challenges: (1) the 128-layer process yields lower density per die compared to Samsung's 236-layer or Micron's 232-layer QLC, resulting in a cost disadvantage at the die level that partially offsets QLC's intrinsic $/TB advantage; (2) enterprise-grade QLC requires mature firmware tuning for wear management, sustained write performance under mixed workloads, and extensive qualification — processes that are still maturing in the YMTC ecosystem.

YMTC's 232-layer QLC is the product generation that could be truly competitive in the data center market, offering die densities of 2 Tbit per die and enabling 16 TB and 32 TB SSDs at compelling cost points. The timeline for volume 232-layer QLC production is the critical variable for China's domestic QLC SSD ambitions.

### 2.2 Other Domestic NAND Efforts

Beyond YMTC, China's domestic NAND landscape is thin:

- **CXMT (ChangXin Memory Technologies):** Based in Hefei, CXMT is China's primary DRAM manufacturer and has no current NAND flash production. However, its existence is relevant to the broader memory ecosystem — CXMT's DRAM capabilities complement YMTC's NAND, and both benefit from overlapping government support and equipment supply chains.
- **GigaDevice and Dosilicon:** These companies produce NOR flash and niche NAND products, but neither operates 3D NAND fabs at the density or scale relevant to data center SSDs.
- **Research initiatives:** Multiple Chinese universities and research institutes (e.g., the Institute of Microelectronics at the Chinese Academy of Sciences) conduct NAND-related research, but no new entrant is close to establishing a second large-scale 3D NAND manufacturing operation. The capital requirements ($10–20 billion for a single competitive fab) and equipment access restrictions make new entry extremely difficult.

**Third Fab Expansion:** YMTC is moving forward with construction of a third manufacturing plant in Wuhan. A new entity, Changcun Phase III (Wuhan) Integrated Circuit Co., Ltd., was established on September 5, 2025 with registered capital of RMB 20.72 billion (~$3 billion), targeting mass production in 2027 ([TrendForce YMTC Phase III](https://www.trendforce.com/news/2025/09/09/news-chinas-ymtc-launches-3b-wuhan-phase-iii-venture-signaling-nand-expansion-ambitions/); [Nikkei Asia YMTC 3rd Plant](https://asia.nikkei.com/business/tech/semiconductors/china-s-top-memory-chip-maker-ymtc-to-build-3rd-plant-eyeing-2027-start)).

**Domestic Equipment Pilot:** Critically, YMTC plans to pilot its first NAND production line built entirely with domestic Chinese tools in H2 2025, achieving a 45% domestic equipment adoption rate — the highest in China's semiconductor industry ([Digitimes YMTC Homegrown Line](https://www.digitimes.com/news/a20250721PD205/ymtc-memory-nand-3d-stacking-2026.html)). However, persistent gaps in extreme-ultraviolet lithography and advanced etch tools mean sustained gains depend on closing the equipment and yield deficit.

In practical terms, YMTC is and will remain China's sole domestic source of enterprise-grade 3D NAND flash through at least 2028. This single-supplier dependency is a strategic vulnerability that China's government and industry are keenly aware of.

---

## 3. Government Policy and Localization

### 3.1 Semiconductor Self-Sufficiency Mandates

China's drive toward semiconductor self-sufficiency is codified in a series of policies spanning more than a decade:

- **National IC Industry Promotion Guidelines (2014):** Established the strategic framework and the first phase of the Big Fund (approximately $21 billion in initial capitalization). YMTC was a direct beneficiary.
- **Made in China 2025 (中国制造2025):** Set targets for domestic content in key sectors, including 70% self-sufficiency in semiconductors by 2025. While this headline target has not been met for advanced chips, it has driven sustained investment and procurement preference policies.
- **14th Five-Year Plan (2021–2025):** Elevated integrated circuits to a top priority under "science and technology self-reliance" (科技自立自强). The plan specifically calls for breakthroughs in memory chips and storage technology.
- **Big Fund Phase II (~$29 billion, 2019) and Phase III (~$47 billion, 2024):** The third phase of the National IC Fund, established in May 2024, was the largest ever — signaling that China is accelerating, not retreating, on semiconductor investment despite export controls. Memory and storage are understood to be among the key investment targets. YMTC's third fab (Phase III, ~$3B registered capital) represents a direct manifestation of this investment strategy ([TrendForce YMTC Phase III](https://www.trendforce.com/news/2025/09/09/news-chinas-ymtc-launches-3b-wuhan-phase-iii-venture-signaling-nand-expansion-ambitions/)).

For the SSD market specifically, these policies translate into:

- **Government procurement preferences:** Central and local government agencies, SOEs, and critical infrastructure operators are strongly encouraged (and in some cases required) to use domestically manufactured SSDs. The China Information Technology Security Evaluation Center (CNITSEC) and the Ministry of Industry and Information Technology (MIIT) maintain catalogs of approved domestic products.
- **Subsidy and tax incentives:** Domestic SSD manufacturers and their NAND suppliers benefit from tax holidays, land grants, and direct subsidies that improve their cost competitiveness against foreign incumbents.
- **"Xinchuang" (信创) initiative:** Short for "Information Technology Application Innovation," this program specifically targets replacement of foreign IT products (servers, storage, networking, operating systems, databases) with domestic alternatives across government and critical industries. SSD localization is a component of Xinchuang.

### 3.2 US Export Controls and Their Impact

The US government has imposed progressively tighter restrictions on China's semiconductor industry, with direct implications for NAND production and the SSD supply chain:

**October 2022 — Bureau of Industry and Security (BIS) Rules:**
The landmark October 7, 2022 export control package restricted the sale of advanced semiconductor manufacturing equipment to China. For NAND specifically, the rules targeted equipment used to produce 128-layer or above 3D NAND flash. Key restricted categories included:
- Advanced deposition tools (e.g., Lam Research etch systems, Applied Materials CVD tools)
- Lithography equipment (ASML, though NAND relies on DUV rather than EUV, certain advanced DUV capabilities were restricted)
- Inspection and metrology tools (KLA Corporation)

**December 2022 — YMTC Entity List:**
YMTC was added to the US Entity List on December 15, 2022, requiring US companies to obtain individual licenses before supplying equipment, software, or technology to YMTC. This effectively cut YMTC off from its primary equipment suppliers. The impact was immediate and severe: YMTC's planned capacity expansion from approximately 100,000 to 200,000+ wafer starts per month was stalled, and the 232-layer ramp was delayed by an estimated 12–18 months.

**October 2023 — Updated BIS Rules:**
The updated rules closed loopholes, expanded coverage to additional equipment categories, and extended restrictions to entities in third countries (notably Japan and the Netherlands) to prevent circumvention through non-US-origin equipment. Japan and the Netherlands implemented corresponding export restrictions on semiconductor equipment (Tokyo Electron, Nikon, ASML, ASM International).

**2024–2025 — Continuing Tightening:**
Additional restrictions in 2024 further constrained the flow of maintenance and servicing support for equipment already installed at YMTC's fabs. The "foreign direct product rule" (FDPR) application was broadened to cover more items.

**Impact on YMTC and the SSD market:**
- **Capacity constraints:** YMTC's total wafer output has plateaued well below its pre-sanctions expansion targets. Without access to additional advanced etch and deposition tools, YMTC cannot build out new production lines or significantly expand existing ones.
- **Technology node limitation:** While YMTC has the technical knowledge to produce 232-layer NAND, scaling it to volume production requires equipment that is now restricted. Industry analysts believe YMTC is repurposing and modifying existing equipment, and potentially sourcing some tools through gray-market channels, but output remains constrained.
- **Domestic equipment development:** China is aggressively developing indigenous semiconductor equipment (NAURA Technology, AMEC, ACM Research), but these tools are currently 1–2 generations behind their US, Japanese, and Dutch counterparts. Domestic etch tools, for example, can handle some NAND production steps but not the most demanding high-aspect-ratio etches required for 200+ layer devices.
- **Net effect on QLC SSDs:** The sanctions have delayed YMTC's QLC roadmap for data center applications by creating a capacity bottleneck. Even if YMTC's 232-layer QLC technology is competitive on paper, the inability to produce it at scale means domestic QLC data center SSDs using YMTC NAND will remain supply-constrained through at least 2026–2027.

### 3.3 Dual Circulation Strategy Implications

President Xi Jinping's "dual circulation" (双循环) economic framework, articulated since 2020, emphasizes development of a robust domestic economic cycle (内循环) while maintaining participation in the international economy (外循环). For the storage industry, this means:

- **Demand-side pull:** Domestic cloud providers and enterprises are incentivized to qualify and adopt domestic SSDs, creating a guaranteed demand floor for companies like Longsys, Biwin, and Memblaze.
- **Supply-side push:** Government investment in YMTC and the domestic equipment ecosystem ensures continued supply development even when it is not yet commercially competitive.
- **Strategic stockpiling:** There is evidence that Chinese companies have built significant inventories of foreign NAND flash and SSDs as a hedge against further export restrictions. Samsung, SK Hynix, and Micron all reported strong China demand through 2023–2024, partly attributed to precautionary buying.

---

## 4. Cloud and Internet Giants

China's cloud and internet companies are the largest SSD consumers in the domestic market. Their procurement strategies directly shape the competitive landscape for QLC SSDs.

### 4.1 Alibaba Cloud (阿里云)

Alibaba Cloud is China's largest cloud provider by revenue (approximately 35.8% market share in China's AI cloud market as of 2025, significantly outperforming ByteDance's Volcano Engine at 14.8%) ([SCMP Alibaba Cloud Lead](https://www.scmp.com/tech/big-tech/article/3325034/alibaba-holds-wide-lead-over-rivals-bytedance-huawei-tencent-chinas-ai-cloud-market)). The Chinese market for AI cloud services more than doubled in 2025 to 51.8 billion yuan (~$7.3 billion), up from 20.83 billion yuan in 2024. Alibaba Cloud is one of the world's largest SSD consumers. Its storage architecture spans multiple tiers:

- **Pangu Distributed Storage System:** Alibaba's proprietary distributed storage platform, analogous to Google's Colossus, underpins EBS (Elastic Block Storage), OSS (Object Storage Service), and NAS offerings. Pangu's architecture separates compute and storage, enabling large-scale SSD deployments for the storage tier.
- **SSD procurement mix:** Alibaba has historically sourced SSDs from Samsung, Intel (now Solidigm/SK Hynix), Micron, and Kioxia, supplemented by custom-spec SSDs from ODMs. Starting around 2022–2023, Alibaba began qualifying domestic SSDs from partners using YMTC NAND for specific tiers — primarily read-heavy workloads and cold/warm storage where QLC's density advantage is most relevant and endurance requirements are lowest.
- **QLC strategy:** Alibaba has been a proactive adopter of QLC in its storage hierarchy. The company's engineering teams have contributed to open-source storage projects (e.g., participation in Linux kernel NVMe and io_uring development) that improve QLC SSD utilization — such as optimized garbage collection hints, write coalescing at the application layer, and workload-aware data placement. For QLC specifically, Alibaba has deployed foreign-sourced QLC SSDs (notably Solidigm D5-P5336 and Samsung PM9A3 QLC variants) in OSS cold storage tiers and is evaluating domestic QLC alternatives as they mature.
- **Domestic NAND adoption:** Alibaba's approach has been pragmatic — qualifying domestic SSDs alongside foreign products and deploying them where performance meets requirements, rather than forcing wholesale substitution. This staged approach is expected to increase domestic SSD penetration to 30–40% of new procurement by 2026–2027, up from an estimated 10–15% in 2024.

### 4.2 Tencent Cloud (腾讯云)

Tencent operates one of the world's largest online services ecosystems (WeChat, QQ, gaming) and a substantial cloud business. Its data center SSD strategy reflects both the consumer internet and cloud infrastructure sides of its business:

- **Storage architecture:** Tencent uses a combination of centralized storage clusters and hyper-converged infrastructure. Its proprietary COS (Cloud Object Storage) and CBS (Cloud Block Storage) services consume large volumes of SSDs.
- **SSD procurement:** Tencent has been a major buyer of Samsung and Micron enterprise SSDs. Like Alibaba, Tencent began evaluating domestic SSD options around 2022, driven by both supply security concerns and government encouragement. Tencent has reportedly qualified SSDs from Longsys (using YMTC 128-layer TLC) for certain internal workloads.
- **QLC adoption:** Tencent's gaming and social media platforms generate vast amounts of user-generated content (images, videos, chat logs) that is accessed frequently in aggregate but infrequently at the individual object level — a classic QLC sweet spot. Tencent has been an early mover in deploying QLC for content distribution and caching tiers. Its massive WeChat Moments and Mini Program ecosystems generate petabytes of warm data daily, well-suited to QLC's read-optimized profile.
- **AI storage demands:** Tencent's Hunyuan large language model program and AI-generated content initiatives are driving additional SSD demand for training data lakes and model serving infrastructure.

### 4.3 Baidu (百度)

Baidu's data center strategy has pivoted significantly toward AI, making it one of the more interesting cases for storage analysis:

- **AI-first infrastructure:** Baidu's ERNIE large language model series and its Wenxin Yige (text-to-image) platform have driven massive investment in GPU-accelerated computing clusters. These clusters require high-performance NVMe SSDs for training data staging, checkpoint storage, and inference caching.
- **Storage requirements:** AI training workloads have distinctive storage patterns — large sequential reads of training data, periodic large sequential writes for checkpoints, and burst random reads for data augmentation. This pattern favors high-capacity NVMe SSDs with strong sequential read performance — a profile that QLC SSDs can serve effectively for the read-dominant phases, complemented by TLC or SLC-cached QLC for checkpoint writes.
- **Kunlun AI chip ecosystem:** Baidu has developed its own Kunlun AI accelerator chips, fabricated by Samsung Foundry. This vertical integration ambition extends to storage: Baidu has explored custom SSD specifications optimized for AI workloads, including large-capacity QLC configurations with firmware-level read optimization.
- **Domestic sourcing:** Baidu has been receptive to domestic SSD adoption and has qualified products from Chinese vendors for non-critical storage tiers. The AI training data pipeline, where data is typically read many times but written once, is a natural candidate for domestic QLC SSDs as they mature.

### 4.4 Huawei Cloud (华为云)

Huawei represents the most aggressive case of full-stack domestic technology adoption, driven by necessity after the company was placed on the US Entity List in May 2019:

- **Full-stack domestic strategy:** Huawei Cloud's infrastructure increasingly runs on domestically designed and manufactured components — Kunpeng (ARM-based) server processors, Ascend AI accelerators, and OceanStor storage systems. Huawei's storage division has been at the forefront of integrating YMTC NAND into enterprise products.
- **OceanStor storage systems:** The OceanStor Dorado all-flash storage arrays and OceanStor Pacific series (for unstructured data) are among China's most widely deployed enterprise storage platforms. Huawei has qualified SSDs using YMTC NAND (both TLC and QLC) for these platforms, and domestic NAND content in OceanStor products has been increasing steadily. The new-generation OceanStor Dorado V7 delivers up to 100 million IOPS with a 10x increase in reliability, and supports S3 as well as file and block protocols, with AI-focused features. OceanStor uses proprietary 1.7-inch form factor SSDs with HiSilicon-designed controllers, accommodating 36 drives in a 2U array, with capacity-optimized NVMe SSDs at 15.36, 30.72, and 61.44 TB ([Blocks & Files Huawei AI Storage](https://blocksandfiles.com/2025/11/03/huawei-develops-new-ai-focussed-all-flash-storage/)).
- **HiSilicon SSD controllers:** Huawei's HiSilicon subsidiary has developed SSD controller ASICs, enabling Huawei to build SSDs with both a domestic controller and domestic NAND — an end-to-end domestic SSD. This vertical integration is unique in the Chinese market and mirrors the self-designed controller strategies of Samsung and Micron in the West. HiSilicon's controllers support NVMe 1.4/2.0 and are understood to include hardware-accelerated LDPC ECC optimized for QLC NAND reliability management.
- **QLC in OceanStor:** Huawei has positioned QLC SSDs in OceanStor Pacific for warm/cold data tiers, particularly for video surveillance, medical imaging, and government data archival — sectors where Huawei has dominant market share in China and where QLC's cost advantage is decisive.
- **Sanctions-driven innovation:** Ironically, US sanctions have accelerated Huawei's drive toward a fully domestic storage stack. While this creates short-term performance and cost penalties (YMTC's 128-layer NAND is less dense than Samsung's latest generation), it also builds long-term supply chain resilience and creates a reference architecture that other Chinese companies can follow.

### 4.5 ByteDance (字节跳动)

ByteDance operates one of the world's largest content platforms (TikTok/Douyin, Toutiao, Xigua Video) and has been on a sustained data center expansion trajectory:

- **Scale of operations:** ByteDance is estimated to operate over 1 million servers globally, with the majority located in China. Its data center footprint expanded aggressively from 2020 to 2025, driven by short-video content growth, AI recommendation systems, and the Doubao large language model.
- **Storage profile:** ByteDance's workloads are heavily read-oriented at scale — video serving, image thumbnails, news article caching, and recommendation model features. This read-heavy profile makes ByteDance one of the most natural large-scale QLC adopters in China.
- **SSD strategy:** ByteDance has historically been a large buyer of Samsung and Solidigm SSDs. The company operates its own hardware design team that specifies custom SSD configurations for its Open Compute Project (OCP)-style servers. ByteDance has been evaluating domestic SSD alternatives and is understood to have conducted qualification testing with Longsys and Memblaze products.
- **AI infrastructure:** ByteDance's investment in AI (both for content recommendation and generative AI via Doubao) is driving a new wave of high-capacity SSD procurement for training data storage and GPU cluster local caching. The company's AI training clusters reportedly consume petabytes of high-speed NVMe storage.
- **QLC fit:** ByteDance's content delivery network (CDN) edge caches and object storage tiers are prime candidates for QLC deployment. The company processes and stores billions of video clips, each typically read hundreds or thousands of times after initial upload — an almost ideal QLC workload pattern.

---

## 5. Domestic SSD Ecosystem

A growing ecosystem of Chinese SSD manufacturers is serving the data center market, combining YMTC NAND with domestic or third-party controllers.

### 5.1 Longsys (江波龙 / Longsys Electronics)

Longsys, publicly listed on the Shenzhen Stock Exchange (301308.SZ), is one of China's largest SSD module makers. Through its subsidiary brands — FORESEE (enterprise/industrial) and Lexar (consumer, acquired from Micron in 2018) — Longsys covers the full spectrum of flash storage products.

- **Enterprise SSD portfolio:** Longsys's FORESEE brand offers NVMe enterprise SSDs using YMTC 128-layer TLC NAND in capacities up to 8 TB. These products target Chinese data center operators, cloud providers, and government/SOE customers.
- **QLC products:** Longsys has announced FORESEE enterprise QLC SSDs using YMTC 128-layer QLC NAND, targeting read-intensive data center workloads. As of 2024–2025, these products are in qualification with several major Chinese cloud operators.
- **Scale advantage:** With annual revenue exceeding RMB 10 billion ($1.4 billion), Longsys has the financial scale to invest in enterprise SSD firmware development, testing infrastructure, and customer qualification programs. The company operates SSD assembly and testing facilities in Shenzhen, Hefei, and overseas.
- **Strategic position:** Longsys benefits from its dual sourcing capability — it can use both YMTC NAND and foreign NAND (Samsung, SK Hynix, Kioxia) depending on customer requirements, giving it flexibility to serve both localization-driven and performance-driven procurement.

### 5.2 Biwin (佰维存储)

Biwin Storage Technology (688525.SH, listed on the Shanghai STAR Market) is another major Chinese SSD manufacturer:

- **Enterprise focus:** Biwin has invested heavily in enterprise SSD development, including custom firmware optimization for data center workloads. The company offers NVMe SSDs using YMTC NAND for the domestic enterprise and cloud market.
- **Partnership with YMTC:** Biwin is one of YMTC's closest downstream partners. The company has access to YMTC's latest NAND generations and collaborates on firmware optimization specific to YMTC's Xtacking architecture.
- **Server OEM relationships:** Biwin supplies SSDs to major Chinese server manufacturers including Inspur, H3C (a subsidiary of Tsinghua Unigroup, now restructured under the New H3C Technologies brand), and Sugon — the Chinese OEMs that collectively dominate China's domestic server market.

### 5.3 Memblaze (忆恒创源)

Memblaze is a Beijing-based company specializing in enterprise NVMe SSDs, with a strong focus on the data center segment:

- **Enterprise-first DNA:** Unlike Longsys and Biwin, which also sell consumer products, Memblaze is focused exclusively on enterprise storage. Its PBlaze series of NVMe SSDs has been deployed by multiple Chinese cloud providers and financial institutions.
- **Controller and firmware expertise:** Memblaze differentiates through firmware optimization rather than relying solely on merchant controllers. The company's engineering team has deep expertise in NVMe protocol optimization, flash translation layer (FTL) design, and workload-specific performance tuning.
- **PBlaze 7 series:** Memblaze's PBlaze 7 is available in two configurations: the 7A40 for Asia/Europe and the 7940 for the US, with the 7A40 utilizing regional/domestic components. Both are PCIe 5.0 (Gen5) in U.2 form factor, available in 3.8, 7.68, and 15 TB ([The SSD Review PBlaze 7](https://www.thessdreview.com/our-reviews/enterprise/memblaze-pblaze-7-7a40-gen5-7-68tb-ssd-review/)). Earlier, Memblaze released the PBlaze6 6531 Series as the first PCIe 4.0 NVMe SSD developed based on YMTC 3D NAND. Both DapuStor and Memblaze are now setting sights on global market expansion with cutting-edge enterprise SSDs ([Software Acquisition: DapuStor and Memblaze](https://www.softwareacquisition.com/dapustor-and-memblaze-set-sights-on-global-market-with-cutting-edge-enterprise-ssds/)).
- **QLC readiness:** Memblaze has been developing QLC-optimized firmware that addresses the key enterprise concerns — sustained write performance management, read latency consistency under mixed workloads, and wear-leveling algorithms optimized for QLC's lower endurance budget. This firmware expertise is critical for making YMTC QLC NAND viable in demanding data center environments.

### 5.4 DERA (得瑞领新)

DERA (Shenzhen Derui Microelectronics) is a smaller but notable player in the domestic enterprise SSD market:

- **Self-designed controllers:** DERA has developed its own SSD controller ASICs, making it one of the few Chinese companies (alongside Huawei/HiSilicon and Maxio/InnoGrit) with domestic controller capability. A fully domestic SSD (Chinese controller + YMTC NAND) is strategically significant for Xinchuang compliance.
- **Government and critical infrastructure:** DERA's products are particularly well-positioned for government, military, and critical infrastructure applications where end-to-end domestic supply chain verification is required.
- **Scale limitations:** DERA's revenue and production scale are significantly smaller than Longsys or Biwin, which limits its ability to compete for large cloud procurement contracts.

### 5.5 Controller Ecosystem

The SSD controller landscape within China includes:

- **Maxio Technology (联芸科技):** Publicly traded (688339.SH), Maxio designs SSD controllers for both consumer and enterprise segments. Its MAP1602 and newer controllers support PCIe Gen4/Gen5 and are used by multiple Chinese SSD brands. Maxio has been developing enterprise-grade controllers with features required for QLC data center SSDs (advanced ECC, power-loss protection, NVMe enterprise feature sets).
- **InnoGrit (得一微电子):** Another domestic SSD controller vendor with enterprise ambitions. InnoGrit's Rainier series controllers support NVMe 2.0 and are designed for data center applications.
- **HiSilicon (海思):** As Huawei's chip subsidiary, HiSilicon develops SSD controllers exclusively for Huawei's internal use (OceanStor products). These are not available on the merchant market but represent the most advanced domestic SSD controller capability.

---

## 6. Supply Chain Risks

### 6.1 Sanctions Escalation Scenarios

The current export control regime has constrained YMTC's expansion but has not halted its existing production. Several escalation scenarios could further disrupt China's domestic SSD supply chain:

- **Scenario 1: Equipment maintenance and spare parts restrictions.** If the US and allied nations further restrict the servicing and maintenance of equipment already installed at YMTC's fabs, YMTC's ability to maintain current production levels could degrade over 12–24 months as tools require parts or refurbishment that cannot be sourced.
- **Scenario 2: Secondary sanctions on downstream partners.** If sanctions were extended to domestic SSD companies (Longsys, Biwin, Memblaze) that use YMTC NAND, it could disrupt the entire domestic SSD value chain, although this would be an unprecedented escalation.
- **Scenario 3: Further NAND-specific technology thresholds.** The current restrictions target 128+ layer NAND production equipment. If thresholds were lowered (e.g., to 96 layers), it could impact a larger portion of YMTC's installed production capacity.
- **Scenario 4: Restrictions on NAND flash imports to China.** While unlikely given the negative impact on Samsung, SK Hynix, and Micron revenues, restrictions on NAND flash sales to China would create a dual supply crisis — constraining both domestic production and foreign imports.

### 6.2 Equipment Availability and Domestic Alternatives

China's ability to develop indigenous semiconductor manufacturing equipment is the single most important variable for the long-term trajectory of domestic NAND production:

- **Current state:** Chinese equipment makers (NAURA, AMEC, ACM Research) can supply a growing share of NAND fab tools, particularly for etch, deposition, and cleaning steps at lower layer counts. Industry estimates suggest domestic equipment can cover 15–25% of the tool requirements for a state-of-the-art NAND fab as of 2025, up from approximately 5–10% in 2020.
- **Gap areas:** The most critical gaps are in advanced etch tools for high-aspect-ratio (HAR) features required for 200+ layer 3D NAND, advanced deposition tools with the uniformity required for hundreds of alternating oxide/nitride layers, and high-end metrology and inspection equipment. These are areas where Lam Research, Applied Materials, Tokyo Electron, and KLA have decades of accumulated process knowledge and intellectual property.
- **Timeline for self-sufficiency:** Optimistic projections suggest China could develop domestic equipment capable of supporting 200+ layer NAND production by 2028–2030, but this assumes sustained investment, successful technology development, and no further tightening of export controls on equipment component supply chains.

### 6.3 Technology Gap Analysis

Despite YMTC's impressive progress, a meaningful technology gap persists relative to the global leaders:

| Metric | YMTC (2025–2026 est.) | Global Leaders (2025–2026) |
|--------|-------------------|-----------------------|
| Max production layer count | 270 layers (volume), 294 layers (ramping) | 321 layers (SK Hynix QLC, mass production), 280 layers (Samsung V9), 430 layers (Samsung V10 in H2 2026) |
| QLC die density | ~1–2 Tbit | 2 Tbit (SK Hynix 321L QLC), 2+ Tbit (Samsung V9 QLC) |
| Monthly wafer capacity | ~130K–150K wspm, targeting 15% global share by end 2026 | Samsung ~600K, SK Hynix ~350K, Micron ~250K wspm |
| Enterprise QLC maturity | Progressing; domestic brands qualifying | Volume production; 122 TB and 245 TB drives shipping/qualifying |
| Advanced interfaces | PCIe Gen4/Gen5 NVMe | PCIe Gen5 NVMe standard; PCIe Gen6 in mass production (Micron 9650) |

This gap has direct implications for QLC SSD competitiveness:
- **Cost per TB:** At 128 layers, YMTC's QLC costs more per raw die gigabyte than Samsung's or Micron's latest-generation QLC, partially negating QLC's cost advantage. The 232-layer generation would close this gap substantially.
- **Performance:** Lower die density means more dies per SSD for equivalent capacity, which can actually improve parallelism and random read performance — but it also increases power consumption and BOM cost (more dies, larger PCB, more controller channels utilized).
- **Endurance:** YMTC's QLC endurance specifications are broadly comparable to competitors at the same layer count. The Xtacking architecture's ability to place more sophisticated ECC logic close to the array may offer a slight endurance advantage, though this has not been extensively validated in long-term field deployments.

---

## 7. Key Takeaways and Outlook

### 7.1 Market Trajectory (2025–2030)

**Near-term (2025–2027):**
- China's data center SSD market will continue growing at 15–20% annually, driven by cloud expansion and AI infrastructure investment.
- QLC adoption will accelerate, primarily using foreign-sourced NAND (Samsung, Micron, SK Hynix) for performance-critical deployments, supplemented by YMTC 128-layer QLC for localization-driven procurement and less demanding tiers.
- Domestic SSD vendors (Longsys, Biwin, Memblaze) will gain market share in the 15–25% range for new enterprise SSD procurement, up from roughly 10% in 2024.
- YMTC's 232-layer production will gradually scale, but will not reach the volumes needed to materially shift the supply balance before 2027 unless there is a significant change in the export control landscape.

**Mid-term (2027–2030):**
- If domestic equipment capabilities advance as projected, YMTC could reach 200+ layer volume production using predominantly domestic tools, unlocking significant capacity expansion.
- QLC SSDs using mature YMTC NAND could become cost-competitive with foreign alternatives, enabling domestic products to serve 30–50% of China's data center SSD market.
- Huawei's full-stack domestic SSD model (HiSilicon controller + YMTC NAND) may become the template for broader industry adoption, especially for Xinchuang-compliant applications.
- The bifurcation of the global SSD market — with a domestic Chinese ecosystem operating partly in parallel with the global one — will become more pronounced.

### 7.2 Strategic Implications for QLC SSD Vendors

**For foreign vendors (Samsung, Micron, SK Hynix, Solidigm, Kioxia):**
- China remains a critical and growing market, but the share available to foreign vendors will shrink over time as domestic alternatives mature.
- QLC SSDs with clear technical superiority (higher density, lower power, advanced features like ZNS/FDP support) will maintain differentiation longest.
- Geopolitical risk requires scenario planning for potential restrictions on NAND sales to China — an unlikely but non-zero tail risk.

**For domestic vendors (Longsys, Biwin, Memblaze, DERA):**
- The domestic market opportunity is large and growing, with government policy providing a demand floor.
- QLC firmware maturity is the key differentiator — the vendor that best optimizes YMTC QLC NAND for enterprise reliability and performance will capture disproportionate market share.
- Investment in enterprise SSD qualification processes, endurance testing infrastructure, and customer engineering support is essential to win Tier 1 cloud contracts.

**For YMTC:**
- The 232-layer production ramp is the single most consequential milestone for the domestic QLC SSD ecosystem. Every quarter of delay extends foreign vendor dominance and increases the total cost of the localization effort.
- QLC-specific engineering (die-level optimization, ECC tuning, qualification support for downstream partners) should be a top priority alongside raw capacity expansion.

### 7.3 The Geopolitical Variable

The trajectory of US-China technology competition is the dominant exogenous variable for this market. Three broad pathways are possible:

1. **Status quo continuation:** Export controls remain at current levels. YMTC slowly scales 232-layer production using a mix of existing and domestically developed equipment. Domestic SSD share grows gradually. Foreign vendors retain majority share through 2028–2029.

2. **Escalation:** Additional restrictions further constrain YMTC's production or target NAND flash imports. This would slow domestic QLC SSD development but paradoxically increase the strategic urgency (and government funding) for equipment self-sufficiency.

3. **Partial relaxation:** A diplomatic thaw leads to selective loosening of export controls, possibly allowing YMTC access to some maintenance services or specific equipment categories. This would accelerate the domestic QLC SSD timeline and potentially create a more competitive market.

Under all three scenarios, the direction of travel is clear: China's data center SSD market is moving toward greater domestic content, and QLC — with its favorable economics for China's dominant read-heavy cloud and content workloads — will be at the center of that transition. The question is not whether, but how quickly, and at what cost.

---

*Note: Market sizing estimates cited in this report are synthesized from industry analyst projections (TrendForce, Yole Group, CCID Consulting) and company disclosures available through early 2025. Forward-looking projections reflect analyst consensus ranges and are subject to revision based on policy developments, technology progress, and market conditions.*
