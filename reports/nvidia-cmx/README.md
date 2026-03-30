# NVIDIA Vera Rubin & CMX Research Reports

Deep technical architecture analysis of NVIDIA's Vera Rubin pod and CMX (Context Memory Platform), focused on storage engineers evaluating AI inference infrastructure.

**Completed:** March 2026 | **Sources:** 70+ inline citations from NVIDIA official sources, industry analysts, and tech media

**Related:** [QLC SSD Research Reports](../README.md) — cross-referenced in Report 5

## Reports

### English (`en/`)

| # | Report | Summary |
|---|--------|---------|
| 1 | [Vera Rubin Pod Architecture Overview](en/01-vera-rubin-pod-architecture.md) | 7 chips, 5 rack-scale systems, memory/bandwidth hierarchy, Blackwell comparison |
| 2 | [CMX Context Memory Platform Deep Dive](en/02-cmx-context-memory-platform.md) | KV cache as AI-native data tier, architecture, data flow, 5x performance claims |
| 3 | [BlueField-4 & DOCA Memos](en/03-bluefield4-doca-memos.md) | DPU as storage processor, NVMe management, KV APIs, software stack |
| 4 | [Interconnects: NVLink 6 & Spectrum-X](en/04-interconnects-nvlink6-spectrum-x.md) | 130 TB/s NVLink, RDMA fabric, CMX data path analysis, latency budget |
| 5 | [SSD & Storage Implications](en/05-ssd-storage-implications.md) | CMX SSD requirements, QLC fit, vendor readiness, market impact, 25-50 EB demand |

### Chinese (`zh/`)

| # | Report | Summary |
|---|--------|---------|
| 1 | [Vera Rubin Pod 架构概述](zh/01-vera-rubin-pod-architecture.md) | 七芯片、五机架系统、内存/带宽层次结构、Blackwell对比 |
| 2 | [CMX 上下文内存平台深度解析](zh/02-cmx-context-memory-platform.md) | KV缓存作为AI原生数据层、架构、数据流、5倍性能提升 |
| 3 | [BlueField-4 与 DOCA Memos](zh/03-bluefield4-doca-memos.md) | DPU作为存储处理器、NVMe管理、KV API、软件栈 |
| 4 | [互联架构：NVLink 6 与 Spectrum-X](zh/04-interconnects-nvlink6-spectrum-x.md) | 130 TB/s NVLink、RDMA网络、CMX数据路径分析、延迟预算 |
| 5 | [SSD 与存储影响分析](zh/05-ssd-storage-implications.md) | CMX SSD需求、QLC适配性、厂商就绪度、市场影响、25-50 EB需求 |
