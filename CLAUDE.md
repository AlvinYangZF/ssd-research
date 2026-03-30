# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is a **research workspace** for storage and AI infrastructure technology analysis, focused on enterprise/data center markets. The output is multi-document research suites in English and Chinese.

## Repository Structure

```
reports/
  en/              # QLC SSD English reports (5 documents)
  zh/              # QLC SSD Chinese translations
  README.md        # QLC SSD master index
  nvidia-cmx/
    en/            # NVIDIA CMX English reports (5 documents)
    zh/            # NVIDIA CMX Chinese translations
    README.md      # NVIDIA CMX master index
docs/
  superpowers/
    specs/         # Design specs
    plans/         # Implementation plans
```

## Research Suites

### QLC SSD Research (`reports/en/` and `reports/zh/`)

| File | Topic |
|------|-------|
| `01-qlc-technology-fundamentals.md` | QLC NAND architecture, endurance, ECC, SLC caching |
| `02-competitive-landscape.md` | NAND makers, controller vendors, data center OEMs |
| `03-us-datacenter-market.md` | US hyperscaler/enterprise adoption, supply chain |
| `04-china-datacenter-market.md` | China domestic ecosystem, YMTC, government policy |
| `05-technology-roadmap-2025-2030.md` | PLC, CXL, ZNS/FDP, 300+ layer NAND, PCIe Gen6 |

### NVIDIA Vera Rubin & CMX Research (`reports/nvidia-cmx/en/` and `reports/nvidia-cmx/zh/`)

| File | Topic |
|------|-------|
| `01-vera-rubin-pod-architecture.md` | 7 chips, 5 rack-scale systems, memory hierarchy |
| `02-cmx-context-memory-platform.md` | KV cache as AI-native data tier, architecture, data flow |
| `03-bluefield4-doca-memos.md` | DPU as storage processor, NVMe management, KV APIs |
| `04-interconnects-nvlink6-spectrum-x.md` | NVLink 6, Spectrum-X, CMX data path analysis |
| `05-ssd-storage-implications.md` | CMX SSD requirements, QLC fit, vendor readiness |

Cross-references link CMX Report 5 back to QLC reports (technology fundamentals, competitive landscape, technology roadmap).

## Key Conventions

- **Dual language**: Every report exists in both `en/` and `zh/`. Chinese translations should be professional/natural, not machine-literal.
- **Research-first**: Reports rely heavily on web research. Each claim should cite sources with inline links where available.
- **Audience**: Technical engineers AND business/product leaders (QLC suite); storage engineers (CMX suite).
- **Quality bar**: Substantial documents (3000-6000 words each) with specific data points, vendor products, and strategic analysis — not generic overviews.

## Workflow

- Start with web research before writing any report section
- Write English version first, then translate to Chinese
- Update the relevant README.md index when adding or modifying reports
- Design specs live in `docs/superpowers/specs/`
- Implementation plans live in `docs/superpowers/plans/`
