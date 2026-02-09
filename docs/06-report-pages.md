# 06 — Report Pages

## Overview

The TRoI report contains **8 pages**. All pages use `displayOption: 1` (fit to page). The report uses two custom visuals: **PowerSlicer** and **textFilter**.

| # | Page Name | Ordinal | Size (W×H) | Status |
|---|-----------|---------|------------|--------|
| 1 | Server Inventory | 0 | 1600×900 | Active |
| 2 | Azure vNet | 1 | 1280×720 | Active |
| 3 | Application Mapping Comparison | 2 | 1280×720 | Active |
| 4 | Asset Presence - WIP | 3 | 1600×900 | Active (WIP) |
| 5 | Project Planning | 4 | 1600×900 | Active |
| 6 | Citrix Published Applications | 5 | 1600×900 | Active |
| 7 | Databases | 6 | 1600×900 | Active |
| 8 | Residual Servers Post AVS Migration | 7 | 1600×900 | Active |

> **[REVIEW]**: The plan indicated Asset Presence - WIP might be hidden (visibility:1), but no `visibility` property was found in the report.json. It appears to be visible but is clearly work-in-progress based on its name.

---

## Custom Visuals

| Visual | Package ID | Used For |
|--------|-----------|----------|
| PowerSlicer | `PowerSlicerA96234ADB8D143D1837FEC426BB34781` | Advanced slicer functionality |
| textFilter | `textFilter25A4896A83E0487089E2B90C9AE57C8A` | Text-based filtering |

---

## Page 1: Server Inventory (ordinal 0)

**Purpose**: The primary landing page — comprehensive server inventory view with KPI cards, slicers, and a detailed data table.

**Size**: 1600×900 | **Background**: #F0F3F7

### Layout

```
┌────────────────────────────────────────────────────────────┐
│ [KPI Cards: device counts, snapshot counts, etc.]          │
│ [Cards + shape background with drop shadow + rounded border│
├────────────────────────────────────────────────────────────┤
│ [Slicers: Location, Device Type, Power State, Network, etc.]│
├────────────────────────────────────────────────────────────┤
│ [Main Table: _All Devices data]                            │
│                                                            │
├────────────────────────────────────────────────────────────┤
│ [Additional info/cards row]                                │
│                                                            │
│ [Second Table: additional detail view]                     │
└────────────────────────────────────────────────────────────┘
```

### Visuals
- **KPI Cards**: Multiple card visuals showing device counts and aggregations from `_All Devices`
- **Slicers**: Dropdown slicers for filtering (Location, Device Type, Power State, Network, etc.)
- **Tables**: tableEx visuals displaying `_All Devices` columns
- **Shapes**: Rounded rectangle shapes with drop shadow as card backgrounds

### Page Filters
- `_All Devices[Network]` (Categorical)
- `_All Devices[VM]` (Categorical)

---

## Page 2: Azure vNet (ordinal 1)

**Purpose**: Azure Virtual Network mapping view.

**Size**: 1280×720

### Visuals
- **Main Table**: Large tableEx visual (1270×603) — likely showing Azure VirtualNetworks data
- **Slicer**: One slicer (300×69)

---

## Page 3: Application Mapping Comparison (ordinal 2)

**Purpose**: Compares application mapping between TRoI data model and the Application Catalogue.

**Size**: 1280×720

### Visuals
- **Cards**: Multiple card visuals showing comparison metrics
- **Slicers**: Filtering controls
- **Main Table**: Large tableEx visual (1240×581) for side-by-side comparison

### Page Filters
- `_All Devices[Device Type]` (Categorical)
- `_All Devices[Application]` (Categorical)

---

## Page 4: Asset Presence - WIP (ordinal 3)

**Purpose**: Work-in-progress page for asset presence analysis across systems.

**Size**: 1600×900

### Visuals
- **Slicer pair**: Two slicers (363×67 each) at top
- **Table**: Large visual (857×67 and 858×66)
- **Cards**: Multiple small card visuals (124×63-64 each) — likely showing "In [System]" flag counts
- **Main Table**: Large tableEx visual (1559×686) showing cross-system presence data

> **[REVIEW]**: This page is marked as WIP. Verify if it should be hidden or if development is complete.

---

## Page 5: Project Planning (ordinal 4)

**Purpose**: Migration project planning view with wave assignments, migration paths, and application-server relationships.

**Size**: 1600×900 | **Background**: #F0F3F7

### Layout

```
┌────────────────────────────────────────────────────────────┐
│ [KPI Cards: Applications, Servers, Apps (Apps Cat)]        │
├────────────────────────────────────────────────────────────┤
│ [Slicers: Application, Server Name, Wave, Migration Path,  │
│  Location, Domain, etc.]                                    │
├────────────────────────────────────────────────────────────┤
│ [Main table: _All Projects Scope data]                     │
│                                                            │
├────────────────────────────────────────────────────────────┤
│ [Secondary table]                                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Visuals
- **KPI Cards**: `Applications` measure, `Servers` measure (from `_All Projects Scope`)
- **Slicers** (8): Application, Server Name (Name), Wave, Expected DC Migration Path, Location, Domain, Type, and more — all dropdown mode
- **Tables** (2): tableEx visuals showing `_All Projects Scope` data

### Page Filters
- `_All Projects Scope[Device Type]` (Categorical)
- `_All Projects Scope[Legal Entity Usage (Application Catalogue)]` (Categorical)
- `_All Projects Scope[Network]` (Categorical)
- `_All Projects Scope[Application]` (Categorical)
- `_All Projects Scope[Type]` (Categorical)

---

## Page 6: Citrix Published Applications (ordinal 5)

**Purpose**: Inventory of Citrix published applications.

**Size**: 1600×900

### Visuals
- **KPI Cards**: Multiple card visuals showing application counts
- **Slicers**: Dropdown slicers for filtering
- **Shape background**: Rounded rectangle with drop shadow
- **Main Table**: Large tableEx (1560×688) showing `Citrix Team Published Applications` data

### Page Filters
- `_All Devices[Network]` (Categorical)
- `_All Devices[VM]` (Categorical)

---

## Page 7: Databases (ordinal 6)

**Purpose**: SQL database inventory consolidated from multiple sources.

**Size**: 1600×900 | **Background**: #F0F3F7

### Layout

```
┌────────────────────────────────────────────────────────────┐
│ [Cards: No. of Applications | No. of Servers |             │
│  No. of Instances | No. of Databases]                      │
├────────────────────────────────────────────────────────────┤
│ [Slicers: Application | Server Name | Instance | Database] │
├────────────────────────────────────────────────────────────┤
│ [Main Table: "Databases" title]                            │
│ Columns: Application, Server Name, Full Instance Name,     │
│ Database Name, SQL Version, SQL Edition, Source,            │
│ Server In TRoI                                             │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Visuals
- **KPI Cards** (4):
  - No. of Applications (`DISTINCTCOUNT(Merged SQL Databases[Application])`)
  - No. of Servers (`DISTINCTCOUNT(Merged SQL Databases[Server Name])`)
  - No. of Instances (`DISTINCTCOUNT(Merged SQL Databases[Full Instance Name])`)
  - No. of Databases (`COUNT(Merged SQL Databases[Database Name])`)
- **Shape**: Rounded rectangle background for card group
- **Slicers** (4): Application, Server Name, Full Instance Name, Database Name — all dropdown with multi-select
- **Main Table**: tableEx titled "Databases" displaying `Merged SQL Databases` columns, sorted by Server Name → Instance → Database

### Page Filters
- `_All Devices[Network]` (Categorical)
- `_All Devices[VM]` (Categorical)

### Visual Filter
- Main table has an inverted filter on `_All Devices[Name]` (hides specific names)

---

## Page 8: Residual Servers Post AVS Migration (ordinal 7)

**Purpose**: Shows servers remaining on-premises after AVS migration, helping identify what still needs to be migrated or decommissioned.

**Size**: 1600×900

### Visuals
- **Slicers**: Multiple slicers (300×66 each) for filtering
- **Cards**: Summary metrics
- **Shape backgrounds**: Visual grouping
- **Main Table**: Large tableEx visual (1560×686)

### Page Filters
- Uses `_All Devices` data with focus on Has Matching AVS Server and Location filters

---

## Common Visual Patterns

### Card Style
All KPI cards use:
- Font size 18 for values
- Font size 10 for category labels
- No title shown
- Grouped behind rounded rectangle shapes with drop shadow (transparency 90%)

### Slicer Style
Most slicers use:
- Dropdown mode
- Multi-select enabled
- Self-filter enabled
- Select all checkbox enabled

### Table Style
Main tables use:
- `tableEx` visual type
- Auto-size column width disabled (fixed widths)
- Title shown with descriptive name
- Background enabled
- Rounded border (15px radius) with theme color
- Drop shadow (transparency 90%)

### Page Background
Pages at 1600×900 use background color `#F0F3F7` (light grey-blue) with 0% transparency.
