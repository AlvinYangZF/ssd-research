# QLC SSD 竞争格局：企业级与数据中心市场

**摘要：** 本报告梳理了截至2025年初，QLC（四层单元）NAND闪存SSD在企业级与数据中心领域的竞争格局。报告涵盖完整的产业价值链——从NAND制造商（Samsung、SK Hynix/Solidigm、Micron、KIOXIA/Western Digital、长江存储）到主控芯片厂商（Phison、Silicon Motion、Maxio及自研方案），再到数据中心OEM采购方（Dell、HPE、Lenovo、浪潮、华为）。分析内容涉及具体产品细节、技术差异化、市场份额变化、价格趋势，以及垂直整合和中国国产化进程的战略影响。

---

## 1. 概述：QLC SSD 产业价值链

企业级QLC SSD生态系统由四个相互依存的层级构成：

1. **NAND闪存制造商** —— 五大厂商（Samsung、SK Hynix、Micron、KIOXIA/WD、长江存储）生产原始QLC NAND晶粒。其堆叠层数、单元架构和良率直接决定了SSD的成本结构和性能基准。
2. **SSD主控芯片厂商** —— 主控芯片负责管理QLC固有的复杂性（ECC纠错、SLC缓存分层、磨损均衡）。独立厂商（Phison、Silicon Motion、Maxio）与Samsung和Micron的自研方案展开竞争。
3. **SSD品牌商/模组组装商** —— 垂直整合厂商（Samsung、Micron、Solidigm、KIOXIA）销售成品硬盘。独立品牌（如威刚ADATA、中国的忆联Memblaze）则采购第三方NAND和主控进行组装。
4. **数据中心OEM/超大规模云厂商** —— Dell、HPE、Lenovo、浪潮和华为对QLC SSD进行认证并部署于服务器和存储平台。超大规模云厂商（AWS、Google、Microsoft、字节跳动、阿里巴巴）通常会定制固件或规格形态。

2024-2025年塑造竞争格局的核心动力来自三大趋势的交汇：(a) QLC在30 TB以上容量段的每GB成本已达到与HDD持平的水平；(b) 200层以上NAND技术为企业级耐久性提供了充足的密度裕量，配合超额配置可满足可靠性要求；(c) 超大规模云厂商对读密集型存储层的需求推动QLC进入主流认证周期。

---

## 2. NAND 制造商

### 2.1 Samsung —— V-NAND 领先与垂直整合

**NAND技术：** Samsung一直保持着全球最大NAND闪存制造商的地位，2025年第三季度全球NAND营收份额约为32.3%，营收约60亿美元（[TrendForce Q3 2025 NAND Revenue](https://www.trendforce.com/presscenter/news/20251203-12813.html)）。Samsung第九代V-NAND（V9）采用280层设计，其QLC版本于2024年9月开始量产，实现了比上一代QLC约86%的比特密度提升和两倍的写入性能（[Samsung V9 QLC](https://news.samsung.com/global/samsung-begins-industrys-first-mass-production-of-qlc-9th-gen-v-nand-for-ai-era)）。然而，由于设计挑战，V9 QLC的全面商用化被推迟至至少2026年上半年（[TrendForce Samsung V9 Delay](https://www.trendforce.com/news/2025/09/16/news-samsung-reportedly-delays-v9-qlc-nand-to-1h26-as-sk-hynix-hits-321-layers-in-august/)）。Samsung正在为其V10（430层）NAND建设产线，目标2026年下半年投产，并已公布2030年实现1000层NAND的技术路线（[TrendForce Samsung 1000 Layers](https://www.trendforce.com/news/2025/02/26/news-samsung-reportedly-targets-1000-layer-nand-by-2030-rolls-out-wafer-bonding-at-400-layers/)）。Samsung采用"后打通道孔"的3D NAND架构，从双层堆叠逐步向三层堆叠演进。

**QLC企业级产品：** Samsung的企业级QLC战略以**PM9D3**和**BM1733a**系列为核心，面向数据中心读密集型工作负载。PM9D3基于第七代V-NAND（176层），提供U.2和E1.S规格，最高容量15.36 TB。Samsung还推出了基于第八代V-NAND的**PM9E3**系列，目标是在单个E1.S硬盘中实现30.72 TB容量，已于2024年底开始向超大规模云厂商送样。

Samsung的垂直整合模式——自主设计NAND、主控（自研Elpis及后续平台）、DRAM和固件——使其在QLC优化体系中拥有无可比拟的掌控力。其自研主控可处理企业级QLC所需的复杂SLC到QLC数据折叠、先进LDPC（低密度奇偶校验码）ECC纠错以及多流写入管理。

**战略定位：** Samsung的规模优势使其在消化QLC良率学习成本方面比小型竞争对手更具效率。不过，该公司在2024年面临了运营挑战，包括据报道第八代V-NAND的良率问题曾一度制约供应。Samsung的数据中心SSD营收得益于与所有美国主要超大规模云厂商和OEM的深入合作，以及与中国云服务商（阿里巴巴、腾讯、百度）的紧密关系。

### 2.2 SK Hynix 与 Solidigm —— 坚定押注 QLC

**NAND技术：** SK Hynix已推进至321层QLC NAND，于2025年底开始量产，是全球首个300层以上QLC实现。321层晶粒实现2 Tb容量和6面设计，写入速度提升56%，功耗效率提升23%（[SK Hynix 321-Layer QLC](https://news.skhynix.com/sk-hynix-begins-mass-production-of-321-layer-qlc-nand-flash/)）。SK集团（SK Hynix + Solidigm）在2025年第二季度全球NAND营收份额约21.1%，创历史新高，第三季度营收约35.3亿美元（[TrendForce Q2 2025 NAND](https://www.trendforce.com/presscenter/news/20250828-12688.html)）。

**Solidigm——QLC专精厂商：** SK Hynix在QLC领域最重要的战略资产是**Solidigm**，该公司源于2021年SK Hynix以90亿美元收购Intel的NAND和SSD业务（2021年底完成交割）。Solidigm继承了Intel深厚的QLC技术积累——Intel是首家大规模量产QLC NAND的企业（2018年），并以D5-P4320及后续D5-P5316开创了QLC数据中心SSD先河。

Solidigm的旗舰企业级QLC产品包括：

- **Solidigm D5-P5336（122.88 TB）** —— Solidigm于2024年底/2025年初将其旗舰QLC SSD扩展至122.88 TB，成为全球最大容量的PCIe SSD。基于192层QLC NAND，支持NVMe 2.0和PCIe Gen4 x4，起售价约12,400美元（约0.10美元/GB）（[TechRadar Solidigm 122TB price](https://www.techradar.com/pro/worlds-largest-ssd-is-on-sale-for-almost-usd12-400-and-yes-it-is-quite-a-bargain-if-you-can-afford-it-of-course)）。该产品面向HDD替代场景，适用于读密集型存储层、冷数据存储和内容分发工作负载。Solidigm路线图计划于2026年底前推出245 TB级SSD（[TechRadar Solidigm 245TB](https://www.techradar.com/pro/solidigm-confirms-245-tb-ssds-set-to-launch-before-end-of-2026)）。
- **Solidigm D5-P5436** —— 下一代产品，基于SK Hynix 321层QLC NAND和PCIe Gen5接口，目标实现更高容量。

Solidigm的整体企业战略建立在一个核心判断之上：QLC NAND将在数据中心取代HDD。该公司明确将QLC定位为"存储级"介质，而非在性能维度上与TLC SSD正面竞争。这一理念体现在Solidigm的研发重点上——专注于优化QLC写放大降低算法、后台SLC到QLC数据折叠，以及混合工作负载下的可预测读取延迟。

**战略定位：** SK Hynix领先的NAND制造能力与Solidigm在QLC领域的主控/固件专长相结合，打造了一个专为QLC企业级SSD调优的垂直整合体系。SK Hynix提供NAND晶粒，Solidigm负责主控、固件和成品硬盘设计。二者联合形成了行业内最深入的QLC技术专精度。

### 2.3 Micron —— CMOS-Under-Array 与成本领先战略

**NAND技术：** Micron在NAND领域的差异化竞争优势在于其**CuA（CMOS-Under-Array，阵列下CMOS）** 架构，即将外围逻辑电路置于NAND单元阵列下方而非旁侧。在同等层数下，该架构可将晶粒密度提升约20-30%，直接转化为更低的每GB成本。Micron的232层NAND（即第五代"B58R"节点）采用CuA方案，于2023年进入量产。Micron占全球NAND营收约22-25%。

2024年，Micron发布了**第二代CuA架构**，采用300层以上设计，目标2025年底投产，有望实现进一步的密度提升。

**QLC企业级产品：** Micron的数据中心QLC产品线包括：

- **Micron 6600 ION** —— 6500 ION的后继产品，基于Micron G9 QLC NAND，支持PCIe Gen5接口。提供30.72 TB、61.44 TB和**122.88 TB**（E3.S规格），计划于2026年上半年推出**245 TB**版本（E3.L规格）。截至2026年初，122 TB和245 TB型号正在多家超大规模云厂商进行认证（[Micron G9 Blog](https://www.micron.com/about/blog/storage/ssd/micron-g9-nand-based-ssds-set-the-pace-for-ai-and-cloud)）。
- **Micron 9650（PCIe Gen6）** —— 全球首款PCIe Gen6数据中心SSD，于2026年2月开始量产，顺序读取速度高达28 GB/s。提供面向AI服务器的液冷E1.S版本（[Micron 9650 Mass Production](https://www.storagenewsletter.com/2026/02/19/micron-9650-pcie-gen6-data-center-ssds-in-mass-production/)）。
- **Micron 7600** —— PCIe Gen5 SSD，具备同级最优延迟和QoS（服务质量）性能，面向主流工作负载，与QLC定位的6600 ION形成互补。

**战略定位：** Micron的CuA架构在QLC领域提供了结构性成本优势，因为QLC较低的利润空间使得晶粒面积效率成为最关键的竞争变量。Micron一直积极倡导QLC替代近线HDD，预计到2025-2026年，综合考虑电力、散热、机架空间和管理成本后，QLC SSD将在TCO（总体拥有成本）层面与20 TB以上HDD具备竞争力。

Micron的自研主控能力（通过收购SSD主控IP和内部团队建设发展而来）降低了对第三方主控厂商的依赖，并实现了NAND固件与主控硬件的深度协同优化。

### 2.4 KIOXIA / Western Digital —— BiCS FLASH 与合资企业

**NAND技术：** KIOXIA（前身为东芝存储器）与Western Digital（WD）运营着长期的NAND闪存合资企业，共享KIOXIA在日本四日市和北上工厂的晶圆制造产线。两家公司均使用KIOXIA的**BiCS FLASH** 3D NAND技术。当前量产节点为**218层BiCS8** NAND，于2024年进入量产。KIOXIA占全球NAND营收约12-14%，WD约10-12%，合计约22-26%。

KIOXIA的BiCS FLASH采用传统的外围旁置布局（不同于Micron的CuA），但已引入横向微缩技术以提升密度。KIOXIA已披露正在开发未来节点的CuA方案。

**QLC企业级产品：**

- **KIOXIA XD7P** —— PCIe Gen5读密集型数据中心SSD，采用BiCS FLASH QLC NAND。XD7P面向超大规模云厂商和云服务商工作负载，提供E1.S和U.2规格，最高容量**30.72 TB**。定位为内容分发、数据湖和分析场景的高性价比读取存储层。
- **KIOXIA XD8系列** —— 基于BiCS8（218层）NAND的下一代QLC数据中心SSD，预计2025年开始送样。
- **Western Digital Ultrastar DC SN655** —— WD的数据中心NVMe SSD产品线包含TLC选项，正在基于相同BiCS FLASH供应开发QLC版本。WD在QLC数据中心产品的品牌推广上比KIOXIA更为保守，但已确认针对读密集型存储层的QLC路线图。

**合资企业动态：** KIOXIA与WD的合资关系造成了独特的竞争格局：两家公司共享NAND晶圆供应，却在SSD市场上形成竞争。这种共享成本结构意味着任何一方都无法在NAND成本上获得持久优势。然而，主控/固件差异化和市场策略各有侧重——KIOXIA聚焦企业级/超大规模云渠道，WD则在消费级和部分OEM服务器渠道更具优势。

外界长期关注的KIOXIA IPO（在2023-2024年多次推迟）以及WD可能的合并/拆分动向增加了战略不确定性。WD于2025年初完成闪存业务（现更名为"SanDisk"）与HDD业务的拆分，成为两家独立上市公司。SanDisk此后在QLC领域大举推进，在FMS 2025上发布了其**UltraQLC**技术平台，展示了256 TB企业级NVMe SSD，采用Direct Write QLC（直写QLC）技术绕过SLC缓存，并通过数据保持优化将刷新周期减少约33%（[SanDisk UltraQLC FMS 2025](https://investor.sandisk.com/news-releases/news-release-details/sandisk-showcases-ultraqlctm-technology-platform-milestone)）。独立运营显然加速了SanDisk在企业级QLC领域的投入。

### 2.5 长江存储（YMTC）—— 中国NAND旗舰，承压前行

**NAND技术：** 长江存储（Yangtze Memory Technologies Co., YMTC）是中国领先的本土NAND制造商，采用自主研发的**Xtacking**架构。Xtacking通过混合键合技术将CMOS逻辑晶圆与3D NAND阵列晶圆进行键合——概念上类似于Micron的CuA，但以晶圆对晶圆键合方式实现，而非单片集成。长江存储的Xtacking 3.0架构用于其**232层NAND**，该产品于2023年完成验证。

**QLC进展：** 长江存储已基于128层（Xtacking 2.0）和232层（Xtacking 3.0）节点生产QLC NAND。但QLC产出量有限，大部分产能集中于利润率更高的消费级SSD市场所需的TLC。长江存储的QLC NAND已出现在部分中国本土企业级SSD中，由**忆联（Memblaze）**、**Shannon Systems（现已并入Samsung中国运营体系）**及其他本土组装商生产。

**美国制裁影响：** 长江存储的发展轨迹因美国出口管制而发生根本性转变：

- **2022年10月：** 美国工业与安全局（BIS）对向中国出口半导体设备实施广泛限制，针对先进NAND生产（128层以上）。
- **2022年12月：** 长江存储被列入实体清单，严重限制其获取美国来源的设备、EDA软件和零部件。
- **后续收紧（2023-2024年）：** 进一步限制扩展至设备维护、备件供应和受控技术的第三国转出口。

实际影响为：长江存储可继续使用现有设备生产（主要涵盖128层和部分232层产线），但在扩大产能或向下一代节点（300层以上）过渡方面面临重大障碍——这些节点需要来自Applied Materials、Lam Research、KLA和ASML的先进沉积、刻蚀和量测设备。据报道，长江存储232层的良率和量产爬坡已受到设备维护限制的制约。

尽管面临上述约束，长江存储仍在积极推进扩产。公司正在武汉建设第三座工厂（三期工程，注册资本约30亿美元），目标2027年量产（[TrendForce YMTC Phase III](https://www.trendforce.com/news/2025/09/09/news-chinas-ymtc-launches-3b-wuhan-phase-iii-venture-signaling-nand-expansion-ambitions/)）。长江存储2025年月产能达到约13万-15万片晶圆（约占全球NAND供应的8%），目标在2026年底达到全球15%的份额。公司已推进至270层（据报道已达294层）NAND，与韩国竞争对手的差距正在快速缩小。关键进展是，长江存储计划于2025年下半年试运行首条完全采用国产设备的NAND产线，国产设备采用率达45%——为中国半导体行业最高水平（[Digitimes YMTC Homegrown Tools](https://www.digitimes.com/news/a20250721PD205/ymtc-memory-nand-3d-stacking-2026.html)）。但由于制裁导致产能持续受限，长江存储在2025年第二季度的市场份额降至5%以下（[Tom's Hardware YMTC Struggles](https://www.tomshardware.com/pc-components/ssds/chinas-premier-memory-maker-ymtc-struggles-amid-chokehold-of-us-sanctions-outdated-chipmaking-tools-and-lack-of-new-tools-has-hampered-production-and-development-of-new-tech)）。

长江存储在中国国内数据中心生态中仍具有核心战略地位。中国政府政策积极鼓励国内OEM（浪潮、华为、曙光、新华三）认证和部署基于长江存储的SSD，形成了受政策保护的细分市场。

---

## 3. 主控芯片厂商

SSD主控芯片负责将主机命令转化为NAND闪存操作，对QLC企业级性能至关重要。QLC较窄的电压裕度（16个电压态 vs. TLC的8个）要求更精密的ECC纠错、更复杂的SLC缓存管理以及更激进的读重试算法。企业级QLC主控还必须支持掉电保护、端到端数据通路保护以及NVMe企业级功能集（命名空间、SR-IOV、多路径）。

### 3.1 Phison —— E26/E28 Max 与企业级扩张

**产品线：** Phison总部位于台湾，是最大的独立SSD主控厂商。其企业级相关主控包括：

- **Phison E26** —— 于2022-2023年推出的PCIe Gen5 x4 NVMe 2.0主控，主要面向高性能消费级SSD，同时也用于专业/轻量级企业应用。E26支持TLC和QLC NAND，最多8个NAND通道和32个CE（片选信号），搭载Phison第四代LDPC ECC引擎和基于Arm Cortex-R5内核的CoXProcessor 2.0架构。
- **Phison E28** —— 基于台积电6nm工艺的PCIe Gen5x4 NVMe 2.0主控，顺序读取速度高达14.9 GB/s，写入14 GB/s，读取IOPS达260万，写入IOPS达300万。E28作为全球首款"6nm AI计算SSD"主控荣获2026年台湾精品金奖，其架构可与GPU协同扩展GPU显存容量，支持本地AI训练（[Phison E28](https://www.phison.com/en/e28)）。Phison的企业级产品线现包括**Pascari X201**（面向AI训练、HPC和高频交易）和**Pascari D201**（面向高密度超大规模和云环境），均支持Gen5（[Phison Pascari](https://www.phison.com/en/category/article/press-releases)）。
- **Phison X2（企业级产品线）** —— Phison专用企业级主控系列，包括面向超大规模云厂商定制SSD项目的**X2 Pascal**平台。X2平台支持Open Channel SSD和FDP（灵活数据放置）模式，这些模式对QLC磨损优化尤为有利。

**QLC优化：** Phison在QLC专项固件IP上投入大量资源，包括自适应SLC缓存分配（根据工作负载模式动态调整以SLC模式运行的NAND比例）、QLC直写（绕过SLC缓存进行顺序写入以降低写放大）以及先进的NAND内RAID技术，为耐久预算较低的QLC块提供数据保护。

**市场地位：** Phison是非垂直整合SSD品牌在消费级和企业级市场的主要主控供应商。在企业级QLC领域，Phison的客户包括威刚ADATA（其企业级SSD产品线）、Seagate（其Nytro NVMe SSD，后Seagate SSD业务调整前）以及多家中国SSD品牌。Phison正致力于凭借X2平台赢得超大规模云厂商的设计订单，与Google和Microsoft等超大规模云厂商的自研主控项目竞争。

### 3.2 Silicon Motion —— SM8366 与数据中心拓展

**产品线：** Silicon Motion（慧荣科技，SIMO）总部位于台湾，是第二大独立SSD主控厂商。该公司历来在消费级/客户端市场占主导地位，近年来正积极向企业级拓展：

- **SM8366** —— Silicon Motion的企业级/数据中心NVMe主控，支持PCIe Gen5 x4。SM8366面向主流数据中心工作负载，最多8个NAND通道，支持TLC和QLC NAND。包含企业级功能：掉电保护、端到端数据保护、NVMe 2.0规范兼容及命名空间管理。Silicon Motion将该主控定位于为OEM和ODM客户（尤其是中国市场客户）构建数据中心SSD。
- **SM2508** —— PCIe Gen5消费级/准专业主控，可作为轻量级企业工作负载的平台。

**QLC企业级战略：** Silicon Motion的QLC企业级策略与Phison有所不同：SIMO侧重于提供交钥匙参考设计和固件堆栈，帮助缺乏深度固件工程能力的SSD品牌（尤其是中国品牌）缩短产品上市时间。这使SIMO成为中国SSD品牌基于长江存储或其他NAND生产企业级QLC硬盘的关键赋能者。

**MaxLinear收购（已终止）：** MaxLinear对Silicon Motion的拟议收购（2022年中宣布）于2023年中因监管挑战终止，SIMO保持独立。这一结果具有战略意义——它使SIMO得以继续向中国客户销售主控，避免了若被美国母公司收购可能面临的出口管制合规复杂性。

### 3.3 Maxio —— 中国本土主控替代方案

**公司概况：** Maxio Technology（联芸科技，前身源自JMicron SSD主控部门重组后的独立实体）总部位于上海，已成长为主要聚焦中国本土市场的重要SSD主控厂商。

**产品线：** Maxio的企业级相关主控包括**MAP1602**（PCIe Gen4）及更新的Gen5方案。虽然Maxio的主控主要服务于消费级和轻量级企业领域，但在中国推进半导体自主可控的背景下，公司正在开发数据中心级主控。

**战略意义：** Maxio的重要性不在于对Phison或Silicon Motion的技术领先，而在于其作为中国本土主控厂商的定位——可与长江存储NAND自然搭配，实现全国产SSD解决方案。中国政府采购政策日益倾向于或强制要求使用国产设计和制造的组件，为基于Maxio主控的企业级SSD创造了受保护的可寻址市场。

Maxio在企业级领域面临技术挑战：其ECC引擎、耐久管理固件和企业级功能集通常被认为落后Phison和Silicon Motion一代。但该公司正在大力投资，对于读密集型QLC工作负载（对主控复杂度的要求低于混合或写密集型工作负载），Maxio的主控可能足以满足中国国内数据中心部署需求。

### 3.4 自研主控 —— Samsung、Micron 及超大规模云厂商

**Samsung：** Samsung为其所有企业级SSD自主设计主控。其主控部门（曾开发消费级**Elpis**主控和专有企业级主控）提供完整的垂直整合方案：Samsung NAND + Samsung主控 + Samsung DRAM + Samsung固件。这种垂直整合使Samsung能够针对自身V-NAND的电压分布和错误特征，协同优化主控的LDPC ECC参数、读重试序列和SLC缓存策略——QLC由于电压裕度更窄，这一优势更为显著。

**Micron：** Micron已建立自研SSD主控能力，尤其面向数据中心产品。Micron 6500 ION采用Micron自主设计的主控而非第三方方案，享有与Samsung同样的协同优化优势。Micron的主控团队通过内部开发与选择性收购主控IP双管齐下建设而成。

**超大规模云厂商定制芯片：** 一个新兴竞争要素是超大规模云厂商自研SSD主控/固件。Google已公开讨论其定制SSD主控项目；Microsoft也披露了Azure存储基础设施的定制固件需求。这些项目有时基于商用主控的修改版（如Phison或Marvell芯片搭配定制固件），有时则是全定制ASIC。针对QLC，超大规模云厂商的定制主控可实现工作负载级优化——例如，Google的数据中心SSD采用应用感知的数据放置策略，大幅降低特定内部工作负载的QLC写放大。

---

## 4. 数据中心 OEM 与采购方

### 4.1 美国/全球 OEM

**Dell Technologies：** Dell自2022年起在其PowerEdge服务器和PowerStore/PowerScale存储平台上认证QLC NVMe SSD。Dell的策略审慎且针对特定工作负载：QLC SSD被认证用于读密集型存储层（启动、内容缓存、数据湖），TLC仍为混合工作负载的标准推荐方案。Dell从Samsung、Solidigm和Micron采购QLC硬盘，在配置器中为客户提供多厂商选择。Dell PowerEdge R760和R770平台支持最高30.72 TB的E1.S和U.2规格QLC硬盘。Dell的企业存储阵列（PowerStore 3200/9200）已开始认证更大容量的QLC硬盘用于冷数据层，尤其面向计划替换SATA SSD或近线HDD的客户。

**HPE（Hewlett Packard Enterprise）：** HPE为其ProLiant Gen11服务器平台和Alletra存储平台认证QLC SSD。HPE是QLC在"价值耐久"（VE）存储层的积极倡导者，将QLC硬盘推广为低于1 DWPD（每日全盘写入次数）读密集型工作负载的高性价比替代方案。HPE从Samsung、Micron和Solidigm采购。HPE的GreenLake云服务也在读密集型工作负载场景中使用QLC硬盘。

**Lenovo：** 联想数据中心集团（现联想基础设施解决方案集团）为其ThinkSystem服务器平台认证QLC SSD，主要从Samsung、Solidigm和KIOXIA采购。联想的QLC认证进度比Dell和HPE稍慢，反映了其更为保守的企业客户群体。然而，联想在中国市场的强势地位使其在全球和中国本土QLC SSD市场之间发挥着桥梁作用。

**美国超大规模云厂商（AWS、Google、Microsoft、Meta）：** 美国超大规模云厂商是企业级QLC采购的最大单一推动力。这些公司的运营规模使得QLC相对TLC即便仅有10-15%的成本节约，也能转化为数亿美元的存储基础设施节省。主要部署模式包括：

- **AWS：** 在S3和EBS存储层中使用QLC SSD，多厂商采购。AWS是大容量（30 TB以上）QLC硬盘的早期积极采用者。
- **Google：** 已公开分享其数据中心QLC SSD部署经验，强调TCO优势和定制固件优化。Google的SSD阵列中包含大量QLC容量，用于存储和缓存层。
- **Microsoft Azure：** 在Azure基础设施中为存储密集型工作负载部署QLC。Microsoft与SSD厂商合作开发针对Azure特定I/O模式优化的QLC定制固件。
- **Meta：** 在存储基础设施中使用QLC SSD，用于读密集型缓存和内容分发（图片、视频）。Meta的冷存储层天然适合大容量QLC。

### 4.2 中国 OEM 与云服务商

**浪潮（Inspur）：** 浪潮是中国最大的服务器制造商（全球服务器市场份额约10-11%），也是企业级SSD的重要采购方。浪潮为其服务器平台同时认证国际品牌（Samsung、Micron、Solidigm、KIOXIA）和国产品牌（基于长江存储）的SSD。在中国"信创"（信息技术应用创新）战略下，浪潮提供搭载全国产SSD选项的服务器配置——长江存储NAND搭配国产主控——面向政府和国有企业客户。浪潮的QLC采纳节奏追随全球趋势，但增加了一个维度：同步推进国际和国产QLC供应链的认证。

**华为（Huawei）：** 华为企业业务生产服务器（FusionServer，企业服务器业务剥离后改为超聚变Xfusion品牌）、存储（OceanStor）和云服务（华为云）。华为是采用国产NAND最积极的中国OEM，驱动因素是美国实体清单限制使其难以获取美国来源的技术。华为OceanStor存储阵列认证基于长江存储的QLC SSD，华为云已在读密集型存储层部署国产QLC。华为还通过旗下海思半导体设计自研SSD主控，有望构建完全国产的SSD方案（长江存储NAND + 海思主控）。

**曙光（Sugon/Dawning Information Industry）：** 曙光是另一家被列入美国实体清单的中国大型服务器厂商，同样优先采购国产SSD。曙光的HPC（高性能计算）和云基础设施在工作负载适合的场景中使用基于长江存储的QLC SSD。

**新华三（H3C/New H3C Technologies）：** 作为紫光集团（长江存储母公司）旗下子公司，新华三与长江存储有直接的集团关联。新华三的服务器和存储平台天然认证基于长江存储的SSD，集团协同关系也有助于其提前获得长江存储最新NAND节点用于企业级SSD开发。

**中国超大规模云厂商（阿里巴巴、腾讯、百度、字节跳动）：** 中国主要云服务商代表着巨大的QLC需求潜力。这些公司采取双源采购策略：

- **国际采购：** Samsung、Micron和Solidigm SSD用于性能关键型存储层，优先采用最先进技术。
- **国产采购：** 基于长江存储的SSD用于政务云分区、数据主权敏感的工作负载，以及QLC读密集型特性与国产NAND能力相匹配的高性价比存储层。

阿里云和字节跳动在评估大容量QLC SSD方面尤为积极，与Solidigm合作测试61.44 TB硬盘，同时也在探索基于长江存储的替代方案。

---

## 5. 竞争态势

### 5.1 市场份额与行业格局

2024年全球NAND闪存营收约650-700亿美元，从2022-2023年的严重低迷中复苏，并在整个2025年延续强劲增长。2025年第二季度，前五大NAND制造商合计营收146.7亿美元，环比增长超过20%（[TrendForce Q2 2025 NAND](https://www.trendforce.com/presscenter/news/20250828-12688.html)）。按营收计算的市场份额（近似值，基于TrendForce截至2025年第三季度数据）：

| 厂商 | NAND营收份额（2025年Q3） | QLC企业级定位 |
|--------|-------------------|------------------------|
| Samsung | ~32.3%（营收约60亿美元） | 全垂直整合；V9 280层QLC（推迟至2026年H1） |
| SK Hynix + Solidigm | 合计约19%（约35.3亿美元） | 最坚定的QLC优先战略；321层QLC已量产 |
| Kioxia | ~15.3%（约28.4亿美元，环比增长33%） | BiCS8 QLC；Q3 2025环比增速最快 |
| Micron | ~13%（约24.2亿美元） | CuA成本优势；G9 QLC NAND；6600 ION 122/245 TB |
| SanDisk（原WD闪存业务） | ~10-12% | 新独立运营；UltraQLC 256TB平台 |
| 长江存储（YMTC） | <5%（估计） | 聚焦中国国内市场；270-294层；受制裁限制 |

*来源：[TrendForce Q3 2025 NAND Flash Revenue](https://www.trendforce.com/presscenter/news/20251203-12813.html)*

企业级SSD市场在2025年第三季度创下纪录，前五大品牌合计营收约65.4亿美元，环比增长28%，受AI需求从训练扩展到推理的推动（[TrendForce Q3 2025 eSSD](https://www.trendforce.com/presscenter/news/20251205-12819.html)）。Samsung以约24.4亿美元领跑（环比增长28.6%），SK集团以约18.6亿美元紧随其后（环比增长27.3%），Micron达到约9.91亿美元（环比增长26.3%）。2025年企业级SSD整体市场规模估计为250亿美元（[Market Report Analytics](https://www.marketreportanalytics.com/reports/enterprise-ssds-393877)）。QLC在NAND中的份额预计将从2023年的12.9%增长至2027年的46.4%，几乎与TLC持平（[Omdia via OSCOO](https://www.oscoo.com/news/qlc-ssds-full-rise-a-new-storage-paradigm-for-the-ai-era-balancing-cost-and-performance/)）。

### 5.2 定价动态：QLC vs. TLC vs. HDD

QLC在数据中心的根本经济逻辑是每GB成本：

| 介质（2024年近似价格） | 美元/TB（硬盘级别） |
|-----|-----|
| 近线HDD（20-24 TB，CMR） | $12-15/TB |
| QLC NVMe SSD（15-61 TB，读密集型） | $40-60/TB |
| TLC NVMe SSD（3.84-15.36 TB，混合用途） | $70-100/TB |
| TLC NVMe SSD（800 GB-3.84 TB，写密集型） | $120-200/TB |

虽然QLC SSD在原始美元/TB指标上仍比近线HDD高3-4倍，但综合考虑功耗（每颗SSD约5-7W vs. 每颗HDD约8-10W，但SSD每瓦IOPS高3-10倍）、机架密度（配备24颗E1.S QLC SSD的1U服务器可容纳750+ TB vs. 24颗HDD约300 TB）、散热成本和降低的管理开销等因素后，读密集型工作负载的实际差距可缩小至1.5-2倍。

QLC NAND价格曾持续下行，但AI驱动的需求在2025年逆转了这一趋势。2025年全年企业级SSD合同价格飙升超过50%，一块30 TB TLC企业级SSD在2025年第二季度约3,062美元，到2026年第一季度涨至9,000-11,000美元——涨幅达257%。TrendForce预测2025年第四季度企业级SSD合同价格将环比上涨25%以上（[TrendForce eSSD Prices Q3 2025](https://www.trendforce.com/presscenter/news/20251205-12819.html)）。QLC硬盘的介质成本仍比大尺寸HDD在美元/GB基础上高5-7倍，但预计到本十年末这一差距将缩小至4-5倍。随着AI推理需求加速，预计2026年将出现QLC专项短缺（[TrendForce QLC Shortage Forecast](https://www.trendforce.com/presscenter/news/20250915-12714.html)）。

### 5.3 垂直整合作为竞争护城河

企业级QLC SSD市场的一个显著特征是垂直整合厂商所持有的优势：

- **Samsung** 掌控NAND、主控、DRAM和固件——实现最紧密的协同优化和最高利润率。
- **Micron** 掌控NAND、主控和固件（DRAM同样自供），CuA提供结构性成本优势。
- **Solidigm**（依托SK Hynix NAND供应）掌控主控、固件和SSD设计，获得母公司保障的NAND供应。
- **KIOXIA** 掌控NAND和SSD设计，但混合使用自研和第三方主控。

非整合厂商（使用Phison或Silicon Motion主控并外购NAND的SSD品牌）面临结构性利润压力：缺乏协同优化优势，在供应紧张周期面临NAND分配风险，竞争主要依赖品牌和渠道实力而非技术差异化。在企业级QLC领域，这一态势尤为严峻——绝大多数企业级SSD营收集中在垂直整合厂商（Samsung、Micron、Solidigm、KIOXIA），独立品牌仅占很小份额。

### 5.4 中国国产化与供应链分化

中美技术限制正在催生分化的企业级SSD供应链：

**全球市场：** Samsung、Micron、SK Hynix/Solidigm和KIOXIA/WD在技术领先性上展开竞争，QLC企业级SSD纯粹以性能、耐久性、容量和成本指标进行评估。

**中国国内市场：** 一个平行生态正在形成——长江存储NAND + 国产主控（Maxio联芸科技，或华为海思及其他国内厂商设计的主控）+ 国产SSD品牌构建起端到端的中国本土供应链。该生态在技术上落后全球领先者1-2代，但受益于：

1. **政策驱动需求：** 政府采购对信创分类项目中国产组件的强制要求。
2. **受保护的市场准入：** 国际厂商向中国敏感基础设施销售面临日益加强的审查和潜在限制。
3. **中国市场内的价格竞争力：** 长江存储的定价历来较为激进（有时低于成本），在政府补贴支持下争取市场份额。

对于企业级QLC而言，中国国内市场既是增长机遇（大规模数据中心建设），也是受限的竞争环境（局限于长江存储可用的NAND节点）。基于长江存储128层NAND的QLC SSD能满足读密集型工作负载需求，但无法达到Samsung、Micron和Solidigm基于200层以上QLC实现的容量规格（单盘30 TB以上）。如果出口管制持续限制长江存储获取先进制造设备，这一差距可能长期存在。

### 5.5 新兴竞争要素

多个因素正在重塑2025年及以后的竞争格局：

- **PCIe Gen5过渡：** 向PCIe Gen5 NVMe的转型将顺序带宽翻倍（高达14+ GB/s），但QLC固有的写入速度限制意味着Gen5接口优势主要体现在读取性能上。能够高效利用Gen5带宽进行QLC读取的主控厂商将获得竞争优势。
- **E1.S和E3.S规格：** 行业正从U.2迁移至E1.S（主流）和E3.S（大容量）企业级SSD规格，由EDSFF（企业与数据中心SSD外形规范）标准化推动。QLC的高容量天然有利于E3.S长规格，其可容纳更多NAND封装。
- **FDP（灵活数据放置）：** NVMe 2.0的FDP功能允许主机将写入定向至特定NAND区域，减少SSD主控的垃圾回收开销和写放大。FDP对QLC尤为有利，降低写放大可直接延长耐久寿命。Solidigm、Samsung和Micron在其最新企业级SSD中均支持FDP。
- **ZNS（分区命名空间）：** 一种替代性的主机管理数据放置方案，同样有利于QLC耐久性。虽然超大规模云厂商表现出兴趣，但ZNS的采用速度慢于FDP，因为它对主机软件栈的改动更大。
- **CXL（Compute Express Link，计算快速链路）：** CXL互联存储/内存是一个新兴层级，未来可能与QLC在内存映射存储场景中产生交集。CXL 2.0/3.0的内存池化技术可能为大容量QLC NAND创造新的部署模式。

---

## 6. 核心要点

1. **企业级QLC三强鼎立：** Samsung、Micron和Solidigm（SK Hynix）是企业级QLC SSD的明确领导者。三者均具备自有NAND供应、自研或深度集成的主控以及与超大规模云厂商的深厚合作关系。KIOXIA/WD是具有竞争力的第四名，但缺乏同等程度的QLC战略决心。

2. **Solidigm是QLC的坚定赌注。** 没有其他公司像Solidigm那样将企业命运全面押注在QLC上。其61.44 TB D5-P5336产品及通向122.88 TB的路线图代表了行业最激进的容量突破，直指HDD替代市场。

3. **Micron的CuA是结构性成本优势。** CuA（阵列下CMOS）在同等层数下提供约20-30%的晶粒密度提升，转化为更低的每GB成本——QLC竞争力最核心的指标。随着QLC持续在成本维度竞争，这一优势将不断放大。

4. **垂直整合在QLC中比TLC更为关键。** QLC更窄的利润空间和更复杂的固件需求，使得只有垂直整合厂商才能实现的协同优化获得更高回报。使用第三方主控的独立SSD品牌在企业级QLC领域面临结构性挑战。

5. **中国国产QLC生态真实存在但受到约束。** 长江存储 + 国产主控能够服务于中国信创强制要求的部署，但美国出口管制限制了长江存储在密度和容量方面与全球领先者竞争的能力。中国国内市场是一个平行的、政策驱动的竞争舞台，而非开放市场。

6. **QLC不是替代TLC，而是替代HDD。** 企业级QLC的竞争定位始终是在存储层级中"低于"TLC SSD，而非作为TLC的替代品。QLC瞄准的是近线HDD当前主导的读密集型、高容量密度存储层。这对SSD厂商而言是TAM（总可寻址市场）的扩展，而非自我蚕食。

7. **超大规模云厂商是需求端的决定力量。** AWS、Google、Microsoft、Meta、阿里巴巴和字节跳动合计贡献了企业级QLC的绝大部分出货量。它们的认证决策、定制固件要求和价格谈判对竞争格局的塑造力，远超传统企业OEM渠道。

8. **价格趋势有利于QLC持续渗透。** 随着QLC SSD价格预计到2026年降至约25-35美元/TB（从2024年的40-60美元/TB），与HDD的TCO交叉点将扩展至更广泛的工作负载范围。每一代NAND技术演进（300层以上、CuA进一步普及）都将扩大这一成本优势。

9. **FDP将成为QLC差异化因素。** FDP（灵活数据放置）的采用正在加速，其写放大降低的收益对QLC有限的耐久预算尤为宝贵。拥有成熟FDP实现的厂商将在企业级QLC部署中获得切实的竞争优势。

10. **主控厂商整合可能加速。** 随着企业级QLC日益被垂直整合厂商主导，独立主控厂商（Phison、Silicon Motion、Maxio）在企业级QLC领域的可寻址市场将收窄——它们的增长机会主要在于赋能中国国产SSD品牌，以及面向利基企业细分市场的非整合厂商。

---

*注：本报告基于公开产品发布、行业分析机构数据（TrendForce、IDC、Yole Intelligence、Forward Insights）、厂商新闻稿及技术演讲（Flash Memory Summit、OCP Global Summit），以及行业媒体（AnandTech、Blocks & Files、StorageNewsletter）。具体数据点已在适当位置标注来源。市场份额数据为近似值，反映截至2024年第四季度/2025年第一季度的最佳可用估计。建议读者参考最新厂商披露和分析师报告以获取最新数据。*
