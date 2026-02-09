# 08 — VMware VMs Check Model

## Overview

The **VMware VMs Check** is a secondary, lightweight Power BI model used for quick cross-referencing of VMware VMs against server name mappings. It is much simpler than the main TRoI model.

## Model Metadata

| Property | Value |
|----------|-------|
| Compatibility Level | 1600 |
| Culture | en-US |
| Source Query Culture | en-GB |
| Time Intelligence Enabled | Yes |
| Power BI Desktop Version | 2.146.1254.0 (25.08) |

---

## Tables (6)

| # | Table | Type | Purpose |
|---|-------|------|---------|
| 1 | VMware VMs | Import (M query) | Full VMware VM inventory (same source as TRoI) |
| 2 | Mapping VM to ServerName | Import (M query) | VM-to-friendly-name mapping |
| 3 | DateTableTemplate_... | Auto-generated | Template for local date tables |
| 4 | LocalDateTable_...485 | Auto-generated | Date table for VMware VMs[PowerOn] |
| 5 | LocalDateTable_...c56a | Auto-generated | Date table for VMware VMs[Creation date] |
| 6 | LocalDateTable_...9dea | Auto-generated | Date table for VMware VMs[Change Version] |

### VMware VMs
- **Source**: Same 3 SharePoint Excel RVTools exports as TRoI (SVCENT0005, AVS UK South, AVS UK West)
- **Sheet**: `vInfo`
- **Processing**: Same as TRoI — combines 3 sources, filters connected VMs, joins with Mapping VM to ServerName, adds Network column, deduplicates by VM
- **Columns**: 85 columns (identical to TRoI VMware VMs table)

### Mapping VM to ServerName
- **Source**: SharePoint Excel — `https://amlinonline.sharepoint.com/.../Mapping-VM-ServerName.xlsx`
- **Columns**: `VM` (text), `Name` (text) — both uppercased
- **Purpose**: Maps VM names to friendly server names

---

## Relationships (4)

| # | From Table | From Column | To Table | To Column | Type |
|---|-----------|-------------|----------|-----------|------|
| 1 | VMware VMs | PowerOn | LocalDateTable_...485 | Date | Date (datePartOnly) |
| 2 | VMware VMs | Creation date | LocalDateTable_...c56a | Date | Date (datePartOnly) |
| 3 | VMware VMs | Change Version | LocalDateTable_...9dea | Date | Date (datePartOnly) |
| 4 | Mapping VM to ServerName | VM | VMware VMs | VM | Standard |

Only relationship #4 is a "real" user-defined relationship. The other 3 are auto-generated date hierarchy relationships.

---

## Report (1 page)

| Page | Display Name | Size | Visuals |
|------|-------------|------|---------|
| 1 | Page 1 | 1280×720 | **Empty** (no visual containers) |

The report page exists but contains no visuals. This model appears to be used primarily for its data model (table + relationship) rather than for visual reporting.

> **[REVIEW]**: This model may be a development/utility model that is queried directly rather than viewed as a report. Consider whether it should be merged into the main TRoI model or maintained separately.
