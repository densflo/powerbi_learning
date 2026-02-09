# 01 — Model Overview

## Purpose

The **True Reflection of Inventory (TRoI)** Power BI model consolidates IT infrastructure inventory data from 15+ enterprise systems into a single unified view. It provides:

- A deduplicated list of all servers and devices across VMware, Azure, Hyper-V, and physical infrastructure
- Cross-system presence checks (is this server in Active Directory? CMDB? Qualys? etc.)
- Application-to-server mapping with migration planning data
- Database inventory across multiple SQL sources
- Network and subnet mapping
- Gap analysis for security and monitoring tools

## Model Metadata

| Property | Value |
|----------|-------|
| Compatibility Level | 1600 |
| Default Data Source Version | powerBI_V3 |
| Culture | en-US |
| Source Query Culture | en-GB |
| Power BI Desktop Version | 2.146.1254.0 (25.08) |
| Time Intelligence Enabled | No |

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          SOURCE SYSTEMS                                  │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────────────┤
│ VMware   │ Azure    │ Active   │ Snow     │ Service  │ SolarWinds,     │
│ (RVTools)│ (API)    │ Directory│ Software │ Now CMDB │ Qualys, SEP,    │
│          │          │          │          │          │ Panaseer, QRadar│
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬─────┴────────┬───────┘
     │          │          │          │          │              │
     ▼          ▼          ▼          ▼          ▼              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     POWER QUERY (M) IMPORT LAYER                        │
│                                                                          │
│  Query Groups:                                                           │
│  ├─ Hosting Platforms          (VMware, Azure, SolarWinds VMs)          │
│  ├─ Asset Management           (AD, ConfigMgr, Snow, AppCat, CMDB)     │
│  ├─ Monitoring Tools           (Qualys, SEP, QRadar, Panaseer, SWO)    │
│  ├─ Mapping                    (10 mapping/lookup tables)               │
│  ├─ Merged                     (IP, Subnets, SQL databases/instances)   │
│  ├─ Network                    (Diamond IP, Network Team)               │
│  ├─ Other Teams                (Citrix, DR Manager, Environments)       │
│  ├─ Functions                  (3 custom M functions)                    │
│  └─ Diagnostics                (3 developer-only diagnostic queries)    │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     DAX CALCULATED TABLE LAYER                           │
│                                                                          │
│  _All Devices          Core UNION of all servers with 30+ calc columns  │
│  _All Projects Scope   GENERATE from _All Devices × Application mapping │
│  _All Applications     GENERATE with End of Life flags, IBS, Owner      │
│     by Server                                                            │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          REPORT PAGES (8)                                │
│                                                                          │
│  Server Inventory │ Azure vNet │ App Mapping │ Asset Presence (WIP)     │
│  Project Planning │ Citrix     │ Databases   │ Residual Servers         │
└──────────────────────────────────────────────────────────────────────────┘
```

## Query Group Organisation

| # | Query Group | Order | Description |
|---|-------------|-------|-------------|
| 1 | Child Queries | 0 | Parent container for all import queries |
| 2 | Child Queries\Resource Management | 0 | Resource management queries |
| 3 | Child Queries\Hosting Platforms | 1 | VMware, Azure, and SolarWinds VM data |
| 4 | Child Queries\Asset Management & Inventory | 2 | AD, ConfigMgr, Snow, Application Catalogue, ServiceNow |
| 5 | Child Queries\Monitoring Tools | 3 | Qualys, SEP, QRadar, Panaseer, SolarWinds Nodes |
| 6 | Child Queries\Mapping | 4 | 10 mapping/lookup tables for normalisation |
| 7 | Child Queries\Merged | 8 | Merged IP addresses, subnets, SQL data |
| 8 | Child Queries\Functions | 9 | 3 custom Power Query (M) functions |
| 9 | Child Queries\Other Teams | 10 | Citrix, DR Manager, Environments Team |
| 10 | Diagnostics | 9 | Developer-only diagnostic queries (local JSON files) |

## Key Design Patterns

1. **UNION + EXCEPT for deduplication**: The `_All Devices` calculated table starts with VMware VMs, VMware Hosts, and Azure VMs as the base, then adds Hyper-V (via SolarWinds VMs), non-hypervisor Snow Computers, and server-type SolarWinds Nodes — using `EXCEPT` to avoid duplicates against the base set.

2. **LOOKUPVALUE for cross-system enrichment**: Most calculated columns in `_All Devices` use `LOOKUPVALUE` to pull data from other source tables (e.g., Operating System from AD, then ConfigMgr, then Snow as fallback).

3. **COALESCE for fallback chains**: Many columns use `COALESCE` to try multiple sources in priority order (e.g., Domain from AD → Snow → SolarWinds).

4. **GENERATE for application explosion**: `_All Projects Scope` and `_All Applications by Server` use `GENERATE` to create one row per server-application combination.

5. **Mapping tables for normalisation**: 10 mapping tables handle name translation, location lookup, VLAN mapping, and application catalogue cross-references.
