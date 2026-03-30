# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is a **research workspace** for QLC SSD technology and marketing analysis, focused on enterprise/data center markets (US and China). The output is a multi-document research suite in English and Chinese.

## Repository Structure

```
reports/
  en/           # English research reports (5 documents)
  zh/           # Chinese translations (mirrored structure)
  README.md     # Master index with summaries and links
docs/
  superpowers/
    specs/      # Design specs and plans
```

## Research Reports

| File | Topic |
|------|-------|
| `01-qlc-technology-fundamentals.md` | QLC NAND architecture, endurance, ECC, SLC caching |
| `02-competitive-landscape.md` | NAND makers, controller vendors, data center OEMs |
| `03-us-datacenter-market.md` | US hyperscaler/enterprise adoption, supply chain |
| `04-china-datacenter-market.md` | China domestic ecosystem, YMTC, government policy |
| `05-technology-roadmap-2025-2030.md` | PLC, CXL, ZNS/FDP, 300+ layer NAND, PCIe Gen6 |

## Key Conventions

- **Dual language**: Every report exists in both `en/` and `zh/`. Chinese translations should be professional/natural, not machine-literal.
- **Research-first**: Reports rely heavily on web research. Each claim should cite sources with inline links where available.
- **Audience**: Dual — technical engineers AND business/product leaders. Technical depth is expected but business context must accompany it.
- **Time horizon**: Forward-looking (2025-2030). Distinguish confirmed roadmaps from analyst projections.
- **Geographic focus**: US and China data center markets specifically.
- **Quality bar**: Substantial documents (3000-6000 words each) with specific data points, vendor products, and strategic analysis — not generic overviews.

## Workflow

- Start with web research before writing any report section
- Write English version first, then translate to Chinese
- Update `reports/README.md` index when adding or modifying reports
- Design specs live in `docs/superpowers/specs/`
