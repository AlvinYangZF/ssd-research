# QLC NAND 技术基础

**面向企业与数据中心决策者的全面技术分析**

## 摘要

QLC（四层单元）NAND 闪存在每个存储单元中存储四位数据，通过区分 16 个不同的电压状态，实现了量产 NAND 技术中最高的位密度。本报告深入分析 QLC 的单元架构、编程机制、耐久性约束、纠错策略、SLC（单层单元）缓存技术以及实际性能表现。随着超大规模数据中心和企业级用户评估将 QLC 应用于读密集型和温数据工作负载，深入理解这些技术基础对于做出正确的采购和架构决策至关重要。本报告涵盖截至 2026 年初的产品和工艺节点信息，引用了来自 Samsung、Micron、SK Hynix、Solidigm（Intel 前 NAND 事业部）、SanDisk/Kioxia 以及长江存储（YMTC）的具体数据。

---

## 1. 引言：QLC 为何关乎存储经济学

数据中心存储的经济性归结于一个简单的等式：每 TB 成本与每 TB 性能的平衡。随着全球数据创建量持续加速增长——IDC 曾预测到 2025 年全球数据量将达 175 ZB——存储架构师面临着在保持足够性能的同时不断降低 $/TB 的巨大压力，且需覆盖日益增多的工作负载类型。

QLC NAND 正面回应了这一压力。通过在每个单元中存储四位数据（相比之下 TLC 为三位，MLC 为两位，SLC 为一位），QLC 在相同芯片面积上可实现比 TLC 约高出 33% 的位密度。这意味着更大容量的驱动器和更低的每 GB 成本。15.36 TB 和 30.72 TB 规格的企业级 QLC SSD 已成为商业化主流产品；61.44 TB 驱动器于 2024 年投入量产；122.88 TB 驱动器于 2025 年初开始出货（[Solidigm 122TB D5-P5336](https://news.solidigm.com/en-WW/243441-solidigm-extends-ai-portfolio-leadership-with-the-introduction-of-122tb-drive-the-world-s-highest-capacity-pcie-ssd/)）。SanDisk 在 FMS 2025 上展示了基于全新 UltraQLC 平台的 256 TB 企业级 QLC SSD，计划于 2026 年初上市（[SanDisk UltraQLC at FMS 2025](https://www.sandisk.com/company/newsroom/press-releases/2025/2025-08-05-sandisk-showcases-ultraqlc-technology-platform-with-milestone-enterprise-ssd-capacity-at-fms-2025)）。TrendForce 报告称，2024 年用于 AI 推理服务器的 QLC 出货量同比增长 400%，达到 30 EB（[TrendForce QLC Shipments 2024](https://www.trendforce.com/presscenter/news/20240423-12122.html)）。在驱动器层面，QLC SSD 的企业批量价格已接近 $0.06-0.08/GB，但由于 AI 驱动的需求激增，2025 年第四季度企业级 SSD 合约价格环比上涨超过 25%（[TrendForce Q3 2025 eSSD](https://www.trendforce.com/presscenter/news/20251205-12819.html)），不断缩小与近线 HDD 的价差，同时提供远超 HDD 的随机读取性能。

然而，QLC 的密度优势伴随着耐久性、写入性能和错误管理方面的工程取舍。这些取舍并非致命缺陷——通过合适的控制器固件、系统架构和工作负载分配，完全可以有效管控。理解这些取舍，正是 QLC 成功部署与部署失败之间的关键分野。

**商业背景：** QLC SSD 并非写密集型工作负载中 TLC 的替代品，而是 HDD 的替代品，以及分层存储架构中 TLC 的补充。QLC 在数据中心的总可寻址市场（TAM）是目前仍在机械硬盘上的海量读密集型和温数据——包括内容分发、AI 推理模型存储、对象存储、分析数据湖，以及仍需亚毫秒级访问的归档层。

---

## 2. 单元架构：四位、十六态

### 2.1 从 SLC 到 QLC：每单元位数的演进

NAND 闪存通过在每个存储单元的浮栅（Floating Gate）或电荷捕获层（Charge Trap Layer）中存储电荷来实现数据存储。电荷量决定了单元的阈值电压（Vt），通过与参考电压的比较来读取数据。

| 单元类型 | 位/单元 | 电压状态数 | 典型 P/E 循环次数 | 相对成本/GB |
|-----------|-----------|----------------|---------------------|-------------------|
| SLC       | 1         | 2              | 50,000–100,000      | 1.0x（基准）   |
| MLC       | 2         | 4              | 3,000–10,000        | ~0.50x            |
| TLC       | 3         | 8              | 1,000–3,000         | ~0.33x            |
| QLC       | 4         | 16             | 800–1,500           | ~0.25x            |

每增加一位/单元，单元需要区分的电压状态数翻倍。QLC 的 16 个状态在成熟工艺节点上的电压间距窄至 200–300 mV，相比之下 TLC 约为 ~500 mV，SLC 则超过 1 V。这一极窄的电压裕度正是 QLC 大多数技术挑战的根本原因。

### 2.2 电荷捕获闪存与浮栅闪存

现代 3D NAND 主要采用两种基本架构：

**浮栅架构（FG）：** 历史上由 Intel/Micron（现为 Solidigm/Micron）在其 3D NAND 中使用。电荷存储在被氧化物包围的导电多晶硅浮栅上。FG 单元具有良好的电荷保持特性和成熟的物理模型，但随着层数增加，单元间干扰（相邻浮栅之间的容性耦合）问题日益突出。

**电荷捕获闪存（CTF）：** 由 Samsung、SK Hynix、Kioxia/WD 和 YMTC 采用。电荷被捕获在氮化硅层（Si3N4）中，而非导电栅极。CTF 在高层数 3D NAND 中具有多项优势：单元间干扰更低（因为电荷被离散地捕获在氮化物中，无法像导体中那样自由移动）、单元堆叠的制造工艺更简单、且更容易扩展至 200 层以上。

从 176 层世代开始，Micron 也转向了电荷捕获替代栅极架构（CMOS-under-array 配合电荷捕获单元），至此行业整体向 CTF 架构收敛。截至 2024–2025 年，几乎所有新的 QLC NAND 量产均采用电荷捕获闪存技术。

**对 QLC 的意义：** 电荷捕获单元相比浮栅单元的单元间干扰更小，这对于区分 16 个紧密排列的电压级别至关重要。然而，CTF 单元可能出现"电荷脱陷"（Charge De-trapping）现象——被捕获的电子随时间逐渐逃逸，导致阈值电压分布逐步偏移。这使得 QLC 的数据保持管理比电压裕度更宽的其他单元类型更加复杂。

### 2.3 3D NAND 层数扩展

QLC 的大规模商用与 3D NAND 层数的不断提升密不可分：

- **96 层（2018–2019）：** 第一代量产 QLC（Intel 660p、Samsung 870 QVO）。相对 TLC 的密度经济性优势有限。
- **128 层（2020–2021）：** 成本结构改善。Micron 的 128 层 QLC 驱动了 Crucial P3 和早期数据中心驱动器。
- **176 层（2021–2023）：** 企业级 QLC 的转折点。Micron 176L、Samsung V-NAND V7、SK Hynix 176L。Solidigm D5-P5316（144L）实现了 30.72 TB。
- **200+ 层（2023–2025）：** Samsung 236 层（V-NAND V8）、Samsung 280 层（V-NAND V9）、Micron 232 层、SK Hynix 238 层、Kioxia/WD 218 层 BiCS8。这些节点的 QLC 单芯片密度超过 1 Tb（128 GB），使 61.44 TB 和 122.88 TB 规格成为可能。
- **300+ 层（2025–2026）：** SK Hynix 于 2025 年底开始量产全球首款 321 层 QLC NAND，单芯片容量达 2 Tb，配备 6 个平面，写入速度提升 56%，功耗效率改善 23%（[SK Hynix 321-Layer QLC NAND](https://news.skhynix.com/sk-hynix-begins-mass-production-of-321-layer-qlc-nand-flash/)）。Samsung 已开始建设 V10 NAND 产线，目标 430 层，预计 2026 年下半年商用出货（[TrendForce Samsung V10](https://www.trendforce.com/news/2024/10/29/news-samsung-reportedly-plans-400-layer-vertical-nand-by-2026-targeting-1000-layer-nand-by-2030/)）。Micron 推出了 G9 NAND（约 276 层），搭载于 6600 ION 的 122.88 TB 版本，并规划了 245 TB 变体（[Micron G9 QLC NAND](https://www.micron.com/products/storage/nand-flash/qlc-nand/g9-qlc-nand)）。这些进展正在推动 100+ TB SSD 的实现，并将进一步降低 QLC 的每位成本。

每次层数增加都通过在同等晶圆面积上分摊更多位来改善 QLC 的经济性，摊薄了固定制造成本（光刻、沉积、刻蚀）。

---

## 3. 读写机制

### 3.1 编程：增量步进脉冲编程（ISPP）

向 QLC 单元写入数据意味着将其阈值电压精确地放置到 16 个目标分布之一。这通过增量步进脉冲编程（ISPP，Incremental Step Pulse Programming）实现：

1. 向选中的字线施加编程电压脉冲（Vpgm）。
2. 执行验证操作，检查单元的 Vt 是否已达到目标电平。
3. 若未达到，则将 Vpgm 增加一个小步长（ΔVpgm，通常为 200–400 mV），并重复此过程。
4. 通过验证的单元将被禁止进一步编程。

对于 QLC，由于 16 个目标分布紧密排列，此过程需要比 TLC 更多的迭代次数。典型的 QLC 编程操作可能需要 20–30+ 次 ISPP 循环，而 TLC 为 10–15 次。这是 QLC 写入延迟显著高于 TLC 的主要原因。

### 3.2 多遍编程

为提高编程精度并减少错误，QLC NAND 通常采用多遍（也称为多步骤或粗调-精调）编程：

- **模糊-精调编程（两遍）：** 最早由 Kioxia/WD 采用。第一遍"模糊"编程将单元粗略编程至大致电压水平，第二遍"精调"编程将每个单元精确调整至目标分布。此方法可减少对邻近单元的编程干扰效应。
- **三遍编程：** 部分厂商采用三阶段方式——粗调、中间调整和最终精调——以实现 QLC 所需的窄 Vt 分布。

多遍编程进一步增加了 QLC 的写入延迟。典型的 QLC 页编程时间（tProg）范围为 **1,000 至 2,000 微秒**（1–2 ms），相比之下 TLC 约为 500–700 us，MLC 约为 200–300 us。相对于 TLC 的 2–4 倍延迟代价是 QLC 写入性能受限的根本来源。

### 3.3 读取操作与参考电压

读取 QLC 单元需要 15 次顺序感应操作（对应 16 个电压状态之间的 15 个边界），相比之下 TLC 需要 7 次，MLC 需要 3 次，SLC 需要 1 次。实际中，优化的读取方案可减少此开销：

- **低页读取** 通常可用更少的感应操作完成（取决于格雷码位分配，需 1–4 次读取）。
- **高页读取** 需要最多的感应操作（多达 7–8 次）。
- 总体而言，完整读取一个 QLC 单元的 4 个页面可能需要 15 次独立的电压比较。

典型的 QLC 读取延迟（tR）为 **75–120 微秒**，而 TLC 为 50–75 us。读取代价远小于写入代价，因为每次感应操作很快（仅需几微秒），但 15 次比较的累积效应仍可被感知。

### 3.4 页类型与数据映射

每个 QLC 单元存储四位数据，通过格雷码（Gray Coding）映射至四个逻辑页：

- **低页（LP, Lower Page）** —— 最低有效位
- **中页（MP, Middle Page）**
- **高页（UP, Upper Page）**
- **顶页（TP, Top Page）** —— 最高有效位

格雷码确保相邻电压状态仅有一位不同，从而在单元电压发生轻微漂移时最大限度地减少位错误数。格雷码映射方案的选择还会影响哪些页面读取更快、可靠性更高，进而影响固件级别的数据放置策略。

---

## 4. 耐久性挑战

### 4.1 编程/擦除（P/E）循环限制

QLC NAND 的额定 P/E 循环次数约为 **800 至 1,500 次**，具体取决于 NAND 世代、厂商和工艺节点。部分针对企业场景优化的 QLC 产品（如 Solidigm D5-P5336）通过高级控制器算法可达 1,500 次。作为参照：

- SLC：50,000–100,000 次 P/E 循环
- MLC：3,000–10,000 次 P/E 循环
- TLC：1,000–3,000 次 P/E 循环
- QLC：800–1,500 次 P/E 循环

这些数值代表原始误码率（RBER，Raw Bit Error Rate）超过驱动器纠错能力以维持可接受的数据完整性的临界点（企业级驱动器通常要求不可纠正误码率 UBER 低于 10^-17）。

### 4.2 氧化层退化机制

每次 P/E 循环都会损伤电子在编程和擦除过程中穿越的隧穿氧化层：

**缺陷生成：** Fowler-Nordheim 隧穿过程中的高能电子（热载流子）在 SiO2 隧穿氧化层和 Si/SiO2 界面处产生缺陷位点（陷阱态）。这些陷阱态会：
- 随时间推移偏移单元的阈值电压（Vt 漂移）
- 增加 Vt 分布的变异性（分布展宽）
- 形成电荷泄漏路径，降低数据保持能力

**电荷脱陷（CTF 特有）：** 在电荷捕获闪存单元中，存储的电荷可能逐渐从氮化硅层脱陷，导致 Vt 随时间向下偏移。脱陷速率取决于温度、时间和先前 P/E 循环的次数。

**界面态生成：** 反复循环在衬底-氧化物界面产生界面态，改变单元的 I-V 特性并增加读取噪声。

对于 QLC 而言，这些机制的影响尤为显著，因为 16 个 Vt 分布之间的裕度极小。即使轻微的氧化层退化也会导致分布展宽和重叠，使误码率急剧上升。在经历 1,000 次 P/E 循环后，QLC 单元的原始 BER 可能比全新单元高出 10–100 倍。

### 4.3 数据保持与温度影响

QLC 的数据保持能力——即数据在断电状态下可读取的时间——受 P/E 循环次数和存储温度的强烈影响。企业级规范通常要求：

- **全新单元（< 100 次 P/E 循环）：** 40C 下保持 1 年（断电状态）
- **寿命末期单元（达到额定 P/E 上限）：** 40C 下保持 3 个月（断电状态）

在高温环境下（55–85C，数据中心常见温度范围），电荷损失加速，保持时间缩短。企业级 QLC SSD 通过以下方式进行补偿：

- **后台巡检读取：** 定期扫描数据，对接近错误阈值的块进行刷新（重写）
- **自适应读取电压跟踪：** 调整参考电压以跟踪 Vt 分布漂移
- **读取回收/数据刷新：** 主动将错误率上升的块中的数据迁移至更新的块

### 4.4 每日驱动器写入量（DWPD）影响

较低的 P/E 循环次数直接转化为有限的驱动器级写入耐久性。DWPD（Drive Writes Per Day，每日驱动器写入量）指标表示在保修期内（通常为 5 年），驱动器每天可被整盘写满的次数。对于企业级 QLC SSD：

- **典型 QLC：** 0.3–1 DWPD（例如 Solidigm D5-P5316：0.58 DWPD；Solidigm D5-P5336：0.58 DWPD，5 年保修）
- **典型 TLC 企业级：** 1–3 DWPD
- **高耐久 TLC：** 3–10 DWPD

一块 15.36 TB 的 QLC 驱动器在 0.5 DWPD 下，五年内每天可承受约 7.68 TB 的写入量——总写入字节数（TBW）约为 14 PB。对于读密集型工作负载（CDN 缓存、AI 模型服务、媒体流、温对象存储等写入比远低于 0.5 DWPD 的场景），这一耐久性完全充足。

---

## 5. 纠错：LDPC 及其演进

### 5.1 QLC 为何需要高级 ECC

QLC 狭窄的电压裕度和较高的原始误码率（RBER）使其比前几代 NAND 需要显著更强的纠错能力。典型 RBER 值如下：

- **SLC：** 10^-6 至 10^-5（每百万位 1 个错误）
- **MLC：** 10^-5 至 10^-4
- **TLC：** 10^-4 至 10^-3
- **QLC（全新）：** ~10^-3
- **QLC（寿命末期）：** 10^-2 至 10^-1（每 100 位 1–10 个错误）

企业级要求为 UBER < 10^-17（每 100 PB 读取不超过 1 个不可纠正错误）。将 RBER 从 10^-2 级别纠正到 UBER 10^-17 的水平，需要 ECC 能够在每个码字中纠正大量错误。

### 5.2 LDPC（低密度奇偶校验码）

现代 QLC SSD 控制器普遍采用 LDPC 码，其纠错性能接近 Shannon 极限。关键特点如下：

**硬判决解码与软判决解码：**

- **硬判决解码** 将每个单元读取为单一电压值，并对每一位做出硬性 0/1 判断。速度快（单次读取），但纠错能力有限。
- **软判决解码** 在略微偏移的参考电压处执行多次读取（通常每次感应操作 3–7 次读取），以获取每一位的可靠性信息（对数似然比）。这些软信息使 LDPC 解码器能够纠正显著更多的错误——通常为硬判决解码纠错能力的 2–3 倍——代价是读取延迟和控制器处理开销增加。

QLC SSD 高度依赖软判决解码，尤其是在 NAND 老化后。典型的读取流程：

1. **第一次尝试：** 硬判决解码。若成功（错误数少），立即返回数据。延迟：~100 us。
2. **第二次尝试（硬判决失败时）：** 软判决解码，额外执行 3 次读取。纠错能力大幅提升。延迟：~300–500 us。
3. **第三次尝试（软判决失败时）：** 扩展软判决解码，执行 7+ 次读取并进行最大迭代 LDPC 解码。延迟：~1–2 ms。
4. **最后手段：** 利用内部奇偶校验进行类 RAID 重建（如可用），或借助主机级冗余恢复。

### 5.3 码率与开销

QLC 使用的 LDPC 码率低于 TLC（即冗余更高）：

- **TLC：** 通常码率约 ~0.9（10% 校验开销）。对于 16 KB 的数据页，约存储 1.6 KB 的 ECC 校验数据。
- **QLC：** 通常码率约 ~0.85–0.88（12–15% 校验开销）。增加的校验数据消耗了 QLC 部分原始容量优势。

企业级 QLC 驱动器的过量配置（OP, Over-Provisioning）通常也高于 TLC 驱动器——往往保留原始容量的 10–28% 用于 ECC、元数据、SLC 缓存和备用块，而企业级 TLC 通常为 7–15%。

### 5.4 内部 RAID / XOR 冗余

许多企业级 QLC SSD 在驱动器内部实现了类 RAID 奇偶校验保护：

- **Solidigm 的 "RAIN"（独立 NAND 冗余阵列）：** 将数据条带化分布在多个 NAND 芯片上并附加 XOR 校验，可在单个芯片完全失效时恢复数据。
- **Samsung 的内部校验方案：** 类似的逐芯片校验保护。
- **Micron 的 FlexRAID：** 在芯片和平面级别提供校验保护。

这种内部冗余对 QLC 尤为重要，因为较高的错误率意味着单个页面或块的故障更加频繁。内部 RAID 在主机级数据保护之下提供了一层安全网。

---

## 6. SLC 缓存：弥合写入性能差距

### 6.1 根本问题

QLC 的原始写入性能——tProg 为 1–2 ms 且需多遍编程——无法满足企业应用对持续写入带宽的期望。直写 QLC 的速度通常在单驱动器上为 **100–400 MB/s**，远低于 NVMe 接口的能力上限。SLC 缓存通过先以 SLC 模式（1 位/单元）写入传入数据，再在后台将其迁移至 QLC 来解决这一问题。

**新兴替代方案——直接 QLC 写入：** SanDisk 在 FMS 2025 上发布的 UltraQLC 平台引入了"Direct Write QLC"技术，完全绕过伪 SLC 缓冲区，在第一遍即以掉电安全的方式直接写入 QLC。该方案消除了折叠（folding）开销及相关的写放大，但需要控制器与固件的深度协同优化（[SanDisk UltraQLC Blog](https://www.sandisk.com/company/newsroom/blogs/2025/inside-ultraqlc-the-enterprise-ssd-platform-engineered-for-ai)）。

### 6.2 静态 SLC 缓存与动态 SLC 缓存

**静态 SLC 缓存：** 固定分配一部分 NAND 永久配置为 SLC。例如，在 15.36 TB 的 QLC 驱动器中，可能永久分配 200–500 GB 作为 SLC 缓存。优点：性能可预测，缓存始终可用。缺点：减少可用 QLC 容量，且缓存大小固定，无法根据工作负载调整。

**动态 SLC 缓存：** 控制器动态地将未使用（空闲）的 QLC 块临时指定为 SLC 块。当驱动器大部分为空时，大比例的写入可在 SLC 模式下吸收。随着驱动器填充率上升，动态 SLC 缓存相应缩小。大多数现代企业级 QLC SSD 使用动态 SLC 缓存，有时配合少量静态预留。

**典型缓存大小：**
- 消费级 QLC SSD：12–72 GB 动态 SLC 缓存（因驱动器容量和填充程度而异）
- 企业级 QLC SSD：100–500+ GB，通常动态调整

### 6.3 折叠：SLC 到 QLC 的数据迁移

"折叠"（Folding）是指控制器从 SLC 模式的单元中读取数据并重新编程至 QLC 模式单元的过程。这是一个后台操作，在以下情况下执行：

- SLC 缓存接近容量上限
- 驱动器处于空闲或低负载状态
- 控制器的垃圾回收需要回收 SLC 块

折叠操作的代价较高：
1. 读取 SLC 数据（快速，每页约 25 us）。
2. 组装 4 个页面的数据（因为 QLC 每单元存储 4 位，4 个 SLC 页面折叠为 1 条 QLC 字线的 4 个页面）。
3. 使用多遍 ISPP 对 QLC 单元进行编程（慢速，每字线 1–2 ms）。
4. 验证 QLC 编程是否成功。
5. 擦除 SLC 源块（擦除时间：每块 3–10 ms）。

折叠过程消耗 NAND 带宽并引入写放大（数据先以 SLC 模式写入一次，再以 QLC 模式写入一次——仅缓存写入的最小写放大系数即为 2 倍，尚未计入垃圾回收的影响）。

### 6.4 性能悬崖

当 SLC 缓存耗尽时——因为持续写入超过了缓存容量且控制器来不及折叠——驱动器必须直接写入 QLC。这会产生急剧的性能下降：

- **突发性能（在 SLC 缓存内）：** 顺序写入 3,000–5,000 MB/s（NVMe Gen4），与 TLC 驱动器相当
- **持续性能（直写 QLC）：** 顺序写入 100–500 MB/s，取决于控制器和 NAND 世代

这一 5–30 倍的性能跌落是 QLC 在基准测试和实际使用中最显著的特征。企业级 QLC 驱动器（如 Solidigm D5-P5316、Samsung PM9D3、Micron 6500 ION）通过以下方式缓解此问题：

- **更大的 SLC 缓存**（通常为总容量的 1–3%，在 30.72 TB 驱动器上即 300–900 GB）
- **更快的控制器**，可在混合工作负载中更积极地进行折叠
- **工作负载感知固件**，在检测到空闲期间优先执行折叠操作
- **写入带宽限流**，限制主机写入速率以防止缓存耗尽

对于匹配度高的工作负载（读密集型，写入突发可被 SLC 缓存容纳），性能悬崖极少出现。这也是为什么在部署 QLC 之前进行工作负载特征分析至关重要。

---

## 7. 磨损均衡、读取干扰与数据保持

### 7.1 磨损均衡

磨损均衡（Wear Leveling）将 P/E 循环均匀分配到所有 NAND 块上，防止某个块过早磨损。由于 QLC 的 P/E 预算有限，这一机制尤为关键：

**动态磨损均衡：** 仅在空闲块之间分配写入。热数据（频繁覆写的数据）在多个块之间轮转。对写密集区域有效，但无法解决冷数据长期占据低循环次数块的问题。

**静态（全局）磨损均衡：** 定期将冷数据从低循环次数的块迁移至高循环次数的块，释放低循环次数的块用于新写入。这对 QLC 至关重要，因为即使少量"热"块在缺乏主动管理的情况下也会迅速耗尽其 P/E 预算。

企业级 QLC 控制器实现了复杂的磨损均衡算法，跟踪每块的 P/E 次数并主动重新平衡数据，使所有块的循环次数保持在较窄的范围内。目标是在达到驱动器额定 TBW 时，所有块的磨损程度大致相同。

### 7.2 读取干扰

读取干扰（Read Disturb）指读取 NAND 块中某一字线（一行单元）时，对邻近未被读取字线上的单元造成轻微 Vt 偏移。这是因为读取操作对未选中字线施加了通过电压（Vpass，通常 6–8V），该电压足以引起缓慢的累积电荷注入。

对于 QLC，读取干扰的影响更为突出，原因如下：
- 窄电压裕度意味着即使很小的 Vt 偏移也可能导致位错误
- 数据中心工作负载可能对相同数据执行数十亿次读取（例如 CDN 缓存中的热门内容）
- QLC 块中每块包含更多页面（通常是 SLC 模式的 4 倍逻辑页），增加了每块承受的读取操作次数

**缓解策略：**
- **读取计数跟踪：** 控制器按块统计读取次数。达到阈值（通常 50,000–200,000 次读取）后，将该块纳入读取回收调度。
- **读取回收（数据刷新）：** 将高读取次数块中的数据读出、纠错后重写至新块，然后擦除原块。
- **自适应 Vpass 优化：** 部分控制器动态调整通过电压，在保持可靠读取的同时最大限度减少干扰。

读取回收消耗 P/E 循环，形成一种取舍：防止读取干扰错误需要付出耐久性代价。对于极端读密集型 QLC 工作负载，读取回收消耗的 P/E 循环必须与主机写入一同纳入预算。

### 7.3 垃圾回收的影响

垃圾回收（GC, Garbage Collection）将部分使用块中的有效数据整合到新块中，然后擦除旧块。GC 引起的写放大（WAF, Write Amplification Factor）对 QLC 的影响尤为显著：

- QLC 块较大（当前 200+ 层 NAND 中通常为 48–96 MB/块，而 TLC 为 12–24 MB，因为 QLC 每字线的页数是 TLC 的 4 倍）。
- 更大的块意味着 GC 过程中需要搬迁更多有效数据，增加了写放大。
- 更高的 WAF 从原本就有限的 P/E 预算中消耗更多循环次数。

企业级 QLC SSD 针对典型企业工作负载的目标 WAF 为 1.5–3 倍，通过以下方式实现：
- 过量配置（更多备用块降低 GC 的频率和强度）
- SLC 缓存中的写入合并（在折叠至 QLC 之前批量处理小写入）
- 主机辅助放置（Zoned Namespaces / ZNS 或 Flexible Data Placement / FDP），通过将数据生命周期与擦除块边界对齐来减少 GC

ZNS 和 FDP 是新兴的 NVMe 标准，允许主机管理数据放置，有望将 QLC 的写放大降至接近 1.0 倍。这对 QLC 耐久性可能具有变革意义——有效地将可用写入寿命延长 2 至 3 倍。

---

## 8. 性能特征

### 8.1 顺序性能

企业级 QLC SSD 提供出色的顺序读取性能，可匹配或接近 TLC 驱动器：

| 指标 | 企业级 QLC（PCIe Gen4） | 企业级 TLC（PCIe Gen4） |
|--------|---------------------------|---------------------------|
| 顺序读取 | 6,000–7,000 MB/s | 6,500–7,000 MB/s |
| 顺序写入（SLC 缓存） | 2,000–5,000 MB/s | 3,000–5,500 MB/s |
| 顺序写入（直写 QLC） | 100–500 MB/s | 不适用（TLC 直写：1,500–2,500 MB/s） |

代表性产品（2024–2025）：
- **Solidigm D5-P5336（122.88 TB，PCIe Gen4）：** 读取 7,000 MB/s，写入 3,300 MB/s（SLC 缓存）。起售价约 $12,400（约 $0.10/GB）（[TechRadar Solidigm 122TB](https://www.techradar.com/pro/worlds-largest-ssd-is-on-sale-for-almost-usd12-400-and-yes-it-is-quite-a-bargain-if-you-can-afford-it-of-course)）。
- **Micron 6600 ION（122.88 TB，PCIe Gen5）：** 基于 G9 QLC NAND，245 TB 变体计划于 2026 年上半年推出（[Micron G9 NAND Blog](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)）。
- **Samsung PM9D3（15.36 TB）：** 读取 6,000 MB/s，写入 2,000 MB/s
- **SanDisk UltraQLC（256 TB，U.2）：** 在 FMS 2025 上发布，采用 BiCS8 QLC CBA NAND 和直接 QLC 写入技术；计划于 2026 年初上市（[SanDisk FMS 2025](https://investor.sandisk.com/news-releases/news-release-details/sandisk-showcases-ultraqlctm-technology-platform-milestone)）。

### 8.2 随机性能

随机 I/O 性能是 QLC 与 TLC 差异更为明显的领域，尤其是写入方面：

| 指标 | 企业级 QLC | 企业级 TLC |
|--------|---------------|---------------|
| 随机读取（4K，QD128） | 900K–1,200K IOPS | 1,000K–1,600K IOPS |
| 随机写入（4K，QD128，SLC 缓存） | 100K–250K IOPS | 200K–400K IOPS |
| 混合随机（70/30 读/写） | 200K–400K IOPS | 350K–600K IOPS |

QLC 在高队列深度下的随机读取性能可达 TLC 的 70–90%，使 QLC 非常适合随机读取工作负载，如数据库索引扫描、键值存储读取和内容服务。

### 8.3 延迟特征

延迟是 QLC 底层物理特性最为直观的体现：

| 操作 | QLC 典型值 | TLC 典型值 | SLC 典型值 |
|-----------|-------------|-------------|-------------|
| 读取（tR） | 75–120 us | 50–75 us | 25–30 us |
| 编程（tProg） | 1,000–2,000 us | 500–700 us | 200–300 us |
| 擦除（tErase） | 10–15 ms | 3–5 ms | 1–3 ms |

**读取尾延迟** 是企业工作负载的关键关注点。当 QLC 读取需要软判决解码时（随 NAND 老化可能性增加），读取延迟从 ~100 us 跳升至 300–500 us 甚至 1+ ms。在 P99.9 和 P99.99 分位上，QLC 驱动器的尾延迟可能比中位数读取延迟高 3–10 倍。

企业级 QLC 控制器通过以下方式管理尾延迟：
- **读取重试优化：** 按块缓存最优读取电压偏移量，最大限度减少软判决解码的需求
- **自适应电压跟踪：** 持续更新参考电压以跟踪 Vt 漂移，使更多读取保持在硬判决纠错范围内
- **优先级仲裁：** 允许紧急主机读取抢占后台操作（GC、折叠）

### 8.4 读写不对称性

QLC 表现出所有 NAND 类型中最为极端的读写性能不对称：

- **读取带宽** 在直写 QLC 时可超过写入带宽 5–15 倍
- **读取 IOPS** 可超过写入 IOPS 4–10 倍
- **读取延迟** 在 NAND 层面比写入延迟低 10–20 倍

这种不对称性并非缺陷——而是一种设计特征，使 QLC 成为读主导型工作负载的理想选择。最成功的企业级 QLC 部署都明确将这种不对称性与工作负载需求进行匹配：

- **CDN 和媒体服务：** 95%+ 读取。QLC 表现卓越。
- **AI/ML 模型服务：** 模型权重写入一次后被读取数百万次。QLC 表现卓越。
- **数据湖/分析：** 一次写入、多次读取模式。QLC 具有成本优势。
- **OLTP 数据库：** 50/50 或更高的写入比例。QLC 不适用；应使用 TLC 或 MLC。

---

## 9. 行业创新动态（2024–2025）

### 9.1 Solidigm（Intel NAND 传承）

Solidigm（2021 年起由 SK Hynix 持有，运营 Intel 的前 NAND 业务）是企业级 QLC 最积极的推动者：

- **D5-P5336（122.88 TB）：** Solidigm 于 2024 年底 / 2025 年初将其旗舰 QLC SSD 扩展至 122.88 TB——发布时为全球最高容量 PCIe SSD——基于 192 层 QLC NAND，支持 NVMe 2.0 和 PCIe Gen4 x4 接口。该驱动器提供"无限写入耐久性"（对于典型读密集型工作负载无 DWPD 上限）。Solidigm 的路线图目标为 2026 年底前推出 245 TB 级 SSD（[TechRadar Solidigm 245TB roadmap](https://www.techradar.com/pro/solidigm-confirms-245-tb-ssds-set-to-launch-before-end-of-2026)）。
- **综合工作负载优化：** Solidigm 的固件包含工作负载感知算法，可检测访问模式并相应优化 SLC 缓存管理、读取电压跟踪和 GC 调度。
- **PLC 演示：** Solidigm 在 Flash Memory Summit 2022 上展示了全球首款 PLC（5 位/单元）SSD 原型，彰显了超越 QLC 的长期技术意图（[Solidigm PLC Demo](https://news.solidigm.com/en-WW/217006-solidigm-demonstrates-world-s-first-penta-level-cell-ssd-at-flash-memory-summit/)）。

### 9.2 Samsung

- **V-NAND V9（280 层 QLC）：** Samsung 于 2024 年 9 月开始量产第九代 V-NAND 280 层 QLC，位密度较上一代 QLC 提升约 86%，写入性能翻倍，读写功耗降低 30–50%（[Samsung V9 QLC Press Release](https://news.samsung.com/global/samsung-begins-industrys-first-mass-production-of-qlc-9th-gen-v-nand-for-ai-era)）。但 TrendForce 报道称，由于设计挑战，V9 QLC 的全面商用推迟至至少 2026 年上半年（[TrendForce Samsung V9 Delay](https://www.trendforce.com/news/2025/09/16/news-samsung-reportedly-delays-v9-qlc-nand-to-1h26-as-sk-hynix-hits-321-layers-in-august/)）。
- **V10（430 层）路线图：** Samsung 正在建设 V10 产线，目标 430 层，采用混合键合和超低温刻蚀技术，预计 2026 年下半年商用出货（[Tom's Hardware Samsung V10](https://www.tomshardware.com/pc-components/ssds/samsung-plans-big-capacity-jump-for-ssds-preps-290-layer-v-nand-this-year-430-layer-for-2025)）。Samsung 还披露了 2030 年实现 1,000 层 NAND 的雄心（[TrendForce Samsung 1000 Layers](https://www.trendforce.com/news/2025/02/26/news-samsung-reportedly-targets-1000-layer-nand-by-2030-rolls-out-wafer-bonding-at-400-layers/)）。
- **PM9D3：** Samsung 面向读密集型数据中心工作负载的企业级 QLC SSD，提供 U.2 和 E3.S 规格，最高 15.36 TB。
- **BM1743：** Samsung 面向超大规模客户优化的企业级 QLC 产品，支持 PCIe Gen5 接口。

### 9.3 Micron

- **G9 QLC NAND：** Micron 于 2025 年推出第九代（G9）QLC NAND，驱动新一代数据中心 SSD 产品线。G9 节点采用 Micron 的 CMOS-under-array 架构，实现业界领先的芯片密度（[Micron G9 QLC NAND](https://www.micron.com/products/storage/nand-flash/qlc-nand/g9-qlc-nand)）。
- **6600 ION：** 6500 ION 的继任者，基于 G9 QLC NAND，采用 PCIe Gen5 接口。提供 30.72 TB、61.44 TB 和 122.88 TB（E3.S）版本，245 TB 变体（E3.L）计划于 2026 年上半年推出。2026 年初，122 TB 和 245 TB 型号正在多家超大规模客户处进行认证（[Micron G9 Blog](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)）。
- **9650（PCIe Gen6）：** Micron 于 2026 年 2 月推出全球首款 PCIe Gen6 数据中心 SSD，顺序读取达 28 GB/s——是 Gen5 的两倍。提供面向 AI 服务器的液冷 E1.S 版本（[Micron 9650 Gen6](https://www.storagereview.com/news/micron-9650-nvme-ssd-enters-mass-production-as-first-pcie-gen6-enterprise-ssd)）。
- **CXL QLC 探索：** Micron 已展示基于 CXL 接口的 QLC NAND 作为内存层级的应用，指向未来 QLC 作为 DRAM 之下的持久化内存/存储层级的架构愿景。

### 9.4 SK Hynix

- **321 层 4D NAND QLC：** SK Hynix 于 2025 年底开始量产全球首款 300+ 层 QLC NAND，单芯片容量达 2 Tb，配备 6 个平面（从此前的 4 个增加）。写入速度提升 56%，读取速度提升 18%，写入功耗效率改善 23%（[SK Hynix 321-Layer QLC](https://news.skhynix.com/sk-hynix-begins-mass-production-of-321-layer-qlc-nand-flash/)）。公司计划先将 321 层 NAND 应用于 PC SSD，随后拓展至企业级 SSD 和智能手机。
- **AI NAND 战略：** SK Hynix 发布了 "AIN"（AI NAND）战略，包括使用 3D QLC NAND 的 "AIN D"（Density）解决方案，目标达到 PB 级驱动器容量。公司正在开发基于 321 层 QLC NAND 的 244 TB 企业级 SSD 产品（[Tom's Hardware SK Hynix AI NAND](https://www.tomshardware.com/pc-components/ssds/sk-hynix-unveils-ai-nand-strategy-including-gargantuan-petabyte-class-qlc-ssds-ultra-fast-hbf-and-100m-iops-ssds-also-in-the-pipeline)）。
- **PE8000 系列：** 面向超大规模读密集型工作负载的企业级 QLC SSD。

### 9.5 SanDisk（原 WD Flash）/ Kioxia

- **BiCS8（218 层）：** Kioxia 和 SanDisk（原 Western Digital 闪存业务，2025 年初独立分拆）的最新一代产品。QLC 模式支持高密度企业级和消费级驱动器。
- **UltraQLC 平台：** SanDisk 在 FMS 2025 上发布了 UltraQLC 技术平台，推出 SN670（128 TB）和 256 TB 企业级 SSD（U.2 规格）。核心创新包括 Direct Write QLC（绕过 SLC 缓存）、动态频率调节（约 10% 的性能提升）以及优化的数据保持策略（刷新周期减少约 33%）（[SanDisk UltraQLC](https://investor.sandisk.com/news-releases/news-release-details/sandisk-showcases-ultraqlctm-technology-platform-milestone)）。计划于 2026 年上半年上市。
- **XD8 系列：** E3.S 规格的企业级 QLC SSD，面向数据中心部署。

### 9.6 YMTC（长江存储）

- **X3-9070（232 层）：** YMTC 的最新一代产品，以 TLC 为主但具备 QLC 能力。由于美国半导体设备出口管制（2022 年 10 月实施），供应主要限于中国国内市场。YMTC 的 QLC 主要用于中国国内的消费级 SSD；企业级 QLC 应用仍处于起步阶段。

---

## 10. 核心要点

### 面向技术工程师

1. **QLC 每单元存储 4 位数据，跨越 16 个电压状态**，电压裕度窄至 200–300 mV。这一紧密间距是 QLC 大多数工程挑战的根源——包括耐久性、错误率和写入延迟。

2. **800–1,500 次 P/E 循环的耐久性** 足以应对读密集型工作负载（0.3–1 DWPD）。请仔细计算实际工作负载的写入量；许多"混合型"工作负载的实际写入量远低于 0.5 DWPD。

3. **基于软判决解码的 LDPC** 是 QLC 的必需技术。需要为高 P/E 次数下软判决读取频率增加导致的尾延迟上升做好预案。应监控 P99.9 读取延迟，而非仅关注平均值。

4. **SLC 缓存大小和折叠行为** 决定了突发写入性能。请使用超过缓存容量的持续写入进行测试，以了解最坏情况下的性能表现。直写 QLC 速度（100–500 MB/s）才是真正的性能下限。

5. **ZNS 和 FDP** 可显著降低 QLC 的写放大，有望将有效耐久性延长 2–3 倍。应在新部署中评估这些接口。

6. **读取干扰是热读数据的真实威胁。** 确保驱动器固件包含读取回收功能，并将相关的 P/E 循环开销纳入预算。

### 面向业务和产品决策者

1. **QLC 在驱动器层面可比 TLC 降低 20–40% 的每 TB 存储成本**，在系统层面的降幅更大——因为更高的单盘容量意味着同等容量所需的驱动器、控制器、线缆和机架空间更少。

2. **QLC 替代的是 HDD，而非 TLC。** 应将 QLC 定位于数据中心 70–80% 的读密集型或温数据：CDN 缓存、AI 模型仓库、对象存储、媒体库和分析数据集。

3. **TCO（总拥有成本）分析有利于 QLC**——将电力、散热和密度因素纳入考量后。一块 E1.L 规格的 61.44 TB QLC SSD 可替代 6–8 块近线 HDD，且功耗更低、机架密度更高、访问速度更快。

4. **耐久性可管控，但必须提前规划。** 一块 30.72 TB 的 QLC 驱动器在 0.5 DWPD 下每天可写入约 15 TB，持续 5 年。如果您的工作负载每盘每天写入量低于此值，QLC 的耐久性不会构成约束。

5. **性能悬崖可以避免**——前提是正确的工作负载放置。切勿将 QLC 用于写密集型工作负载。应采用分层架构：TLC/Optane 用于写密集层，QLC 用于读密集和容量层。

6. **200+ 层世代（2024–2025）是企业级 QLC 的转折点。** 容量、成本和可靠性已成熟到足以支撑主流数据中心部署。延迟采用 QLC 的企业，可能在存储成本上相对战略性采用 QLC 的竞争对手处于不利地位。

---

## 参考文献与引用来源

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

*本报告于 2026 年 3 月编制并更新。技术规格反映截至 2026 年初已发布或出货的产品。未来路线图信息（400+ 层 NAND、PLC）基于厂商披露和分析师预测，可能发生变化。*
