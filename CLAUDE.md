# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **True Reflection of Inventory (TRoI)** Power BI project — an enterprise IT infrastructure inventory model that consolidates data from 15+ source systems (VMware, Azure, Active Directory, ServiceNow CMDB, Snow Software, SolarWinds, Qualys, etc.) into a unified server/device view. A secondary lightweight model (**VMware VMs Check**) is also present.

The model uses **import mode** (no DirectQuery) with TMDL (Tabular Model Definition Language) format. All data sources are SharePoint-hosted Excel/CSV files, except one internal web-scraped HTML table.

## Repository Structure

```
True Reflection of Inventory (TRoI).pbip          # Project entry point
True Reflection of Inventory (TRoI).SemanticModel/
  definition/
    model.tmdl              # Model metadata, 9 query groups, 61 table refs
    relationships.tmdl      # 22 relationships (21 active, 1 inactive)
    expressions.tmdl        # 3 M functions, 1 parameter, 1 expression-table, 3 diagnostics
    tables/*.tmdl           # 61 table definitions (one file each)
    cultures/en-US.tmdl     # Localisation
  DAXQueries/
    _All Devices.dax        # Development DAX query (may differ from live partition)

True Reflection of Inventory (TRoI).Report/
  report.json               # 8 report pages, visuals, layout (~62K tokens, very large)

VMware VMs Check.pbip                              # Secondary model entry point
VMware VMs Check.SemanticModel/                    # 6 tables, 4 relationships, 1 empty report page
VMware VMs Check.Report/

docs/                       # Generated documentation (01-09)
```

## Architecture — Three-Layer Design

### Layer 1: Power Query (M) Import
58 source tables imported via M queries, organized into query groups:
- **Hosting Platforms** — VMware (RVTools exports from 3 vCenters), Azure (CSV), SolarWinds VMs
- **Asset Management** — AD, ConfigMgr, Snow, Application Catalogue (parameterized path), ServiceNow CMDB
- **Monitoring Tools** — Qualys, SEP, QRadar, Panaseer, SolarWinds Nodes
- **Mapping** — 10 lookup/normalization tables (some embedded as Base64 JSON, some from SharePoint)
- **Merged** — Combined IP addresses, subnets, SQL databases/instances
- **Network** — Diamond IP, Network Team data
- **Other Teams** — Citrix, DR Manager, Environments Team
- **Functions** — `ExpandSubnet`, `ConvertToMACAddress`, `ConvertIPToDecimal`

### Layer 2: DAX Calculated Tables
Three calculated tables built on imported data:
- **`_All Devices`** — Core UNION of all servers using `EXCEPT` for deduplication. Priority: VMware VMs + Hosts + Azure VMs (base), then Hyper-V via SolarWinds, non-hypervisor Snow Computers, server-type SolarWinds Nodes. Contains 30+ calculated columns using `LOOKUPVALUE`/`COALESCE` fallback chains and 10 measures.
- **`_All Projects Scope`** — `GENERATE` from `_All Devices` x `Mapping Server to Application` (excludes AVS/New ALZ locations and "Client Endpoint")
- **`_All Applications by Server`** — Similar GENERATE but includes all locations, adds End of Life flag for Windows 2008/2012

### Layer 3: Report Pages (8)
Server Inventory | Azure vNet | Application Mapping Comparison | Asset Presence (WIP) | Project Planning | Citrix Published Applications | Databases | Residual Servers Post AVS Migration

## Critical Design Patterns

**UNION + EXCEPT deduplication** — `_All Devices` adds source systems incrementally, using `EXCEPT` against the accumulated set to avoid duplicates. The final dedup filter keeps VM-named rows over BLANK VM rows for the same server Name.

**LOOKUPVALUE fallback chains** — Most `_All Devices` calculated columns try multiple source tables in priority order (e.g., OS: AD -> ConfigMgr -> Snow -> SolarWinds).

**`_All Devices` has NO model relationships** — All cross-table lookups are done via `LOOKUPVALUE` in DAX calculated columns, not through the relationship model.

**Relationship clusters** — VMware detail tables -> `VMware Hosts` (via Host), SQL tables -> `Azure Arc SQL Instances` (via Full Instance Name), Network/Subnet tables -> `Network Team Global IP Address Allocation` (via Subnet).

## Key Gotchas

- **`.dax` file vs partition source** — `_All Devices.dax` comments out `SnowComputersFiltered` in the UNION; the live partition in `_All Devices.tmdl` includes it. The `.dax` file is a development version only.
- **Diagnostics expressions** — 3 queries in `expressions.tmdl` reference `C:\Users\GDVSHB\...` (a previous developer's machine). They will fail on other machines and can be ignored or removed.
- **Ops Theater web scraping** — `Ops Theater RAETSMARINE Apps` scrapes `http://spdoca0001.raetsmarine.local/servers_func.html`. Requires intranet access; fragile to HTML changes.
- **Location column** — ~140 lines of deeply nested IF/LOOKUPVALUE with a 5-tier fallback chain duplicated across 3 branches. Functional but fragile to modify.
- **`ApplicationCataloguePath` parameter** — Controls the SharePoint URL for 4 Application Catalogue tables. Update this when the file moves.
- **report.json is ~62K tokens** — Reading the full report file will consume significant context. Target specific pages by searching for their `displayName`.

## Working with TMDL Files

Each `.tmdl` file is a plain-text definition. Key syntax:
- `table 'Name'` — Table definition with columns, measures, partitions
- `column 'Name' = <DAX>` — Calculated column
- `measure 'Name' = <DAX>` — Measure
- `partition 'Name' = m` — M (Power Query) source; `= calculated` for DAX calculated tables
- `ref table 'Name'` — Table reference in model.tmdl
- `relationship <id>` — Relationship definition in relationships.tmdl
- Annotations starting with `PBI_` are Power BI metadata

## Extending the Model

**Adding a new source system**: Import via Power Query -> optionally add to `_All Devices` UNION (with `EXCEPT` dedup) -> add `In [System]` boolean column -> add `Not In [System]` gap measure -> update report visuals. See `docs/09-maintenance-guide.md` for DAX patterns.

**Adding a calculated column to `_All Devices`**: Use `LOOKUPVALUE` for single values, `COALESCE` for fallback chains, `CONCATENATEX` for multi-value, `IF(NOT(ISBLANK(LOOKUPVALUE(...))), TRUE, FALSE)` for boolean flags.
