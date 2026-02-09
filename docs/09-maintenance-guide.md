# 09 — Maintenance Guide

## Data Refresh

### Overview

The TRoI model uses **import mode** for all data. This means data must be manually refreshed (or scheduled) to stay current. There is no DirectQuery or live connection.

### Source Dependencies

| Dependency | Type | Location | Notes |
|-----------|------|----------|-------|
| SharePoint Online | Primary | `amlinonline.sharepoint.com` | Most source files are Excel workbooks on SharePoint |
| Application Catalogue | SharePoint Excel (parameter) | `DataCentreStrategy` SharePoint site | Path controlled by `ApplicationCataloguePath` parameter |
| Ops Theater | Internal web server | `http://spdoca0001.raetsmarine.local/servers_func.html` | HTML table scraping — requires intranet access |
| Discovery Data Model | SharePoint folder | `DCExit-MigrationTeam` SharePoint site | Contains RVTools exports and most source Excel files |

### Refresh Steps

1. **Ensure SharePoint access**: Verify you have access to both SharePoint sites:
   - `https://amlinonline.sharepoint.com/sites/DCExit-MigrationTeam/`
   - `https://amlinonline.sharepoint.com/sites/DataCentreStrategy/`

2. **Ensure intranet access**: The Ops Theater web scraping source requires access to `spdoca0001.raetsmarine.local` on the corporate network.

3. **Update source data**: Before refreshing the model, ensure source Excel files have been updated with current data (e.g., new RVTools exports uploaded to SharePoint).

4. **Refresh in Power BI Desktop**: Open the model and click Refresh All, or refresh individual tables/query groups as needed.

5. **Credential prompts**: On first refresh or after credential expiry, Power BI will prompt for SharePoint and web credentials. Use your corporate credentials.

---

## Key SharePoint Dependencies

### Discovery Data Model Folder
**Path**: `Enterprise Apps DC Exit/Power BI Data Models/Discovery Data Model/`
**Site**: `DCExit-MigrationTeam` SharePoint

Contains:
- `RVTools_export_all_SVCENT0005.xlsx` — On-premises VMware data
- `RVTools_export_all_AVSUKSouth.xlsx` — AVS UK South VMware data
- `RVTools_export_all_AVSUKWest.xlsx` — AVS UK West VMware data
- `Mapping-VM-ServerName.xlsx` — VM to server name mapping
- Various other Excel exports from enterprise systems

### Application Catalogue
**Path**: Controlled by `ApplicationCataloguePath` parameter
**Site**: `DataCentreStrategy` SharePoint

The Application Catalogue is a single Excel workbook with multiple sheets:
- Working Catalogue
- Wave Plan
- Discrepancies
- Missing From Data Model

To change the file location, update the `ApplicationCataloguePath` parameter in Power Query.

---

## Adding New Source Systems

### Step 1: Create the Import Query
1. In Power Query Editor, create a new query to connect to the source
2. Apply necessary transformations (type changes, column renames, filtering)
3. Assign the query to the appropriate query group (e.g., `Child Queries\Monitoring Tools`)

### Step 2: Add to `_All Devices` (if it's a device/server source)
1. If the new source contains server/device data that should appear in `_All Devices`:
   - Modify the `_All Devices` partition source DAX
   - Add a new `EXCEPT` variable (like `SolarWindsVMsFiltered`) to add unique devices
   - Include the new variable in the `CombinedDevices` UNION
2. Add appropriate filters to avoid duplicating devices already in the base set

### Step 3: Add Calculated Columns (if enriching `_All Devices`)
1. Add a new `In [System]` boolean column following the existing pattern:
   ```dax
   IF(NOT(ISBLANK(LOOKUPVALUE('<NewTable>'[Name], '<NewTable>'[Name], [Name]))), TRUE, FALSE)
   ```
2. Add any other enrichment columns (e.g., status lookups)

### Step 4: Add Measures (if gap analysis needed)
1. Add a "Not In [System]" measure:
   ```dax
   VAR NotInCount = CALCULATE(COUNTA([In NewSystem]), [In NewSystem] IN {FALSE})
   RETURN IF(NotInCount = BLANK(), 0, NotInCount)
   ```

### Step 5: Update Report Pages
1. Add the new columns/measures to relevant report page visuals
2. Consider adding new slicers if the source provides useful filter dimensions

---

## Adding Calculated Columns to `_All Devices`

The most common maintenance task is adding a new calculated column to `_All Devices`.

### Pattern: Simple LOOKUPVALUE
```dax
column 'New Column Name' =
    LOOKUPVALUE('SourceTable'[ColumnName], 'SourceTable'[Name], [Name])
```

### Pattern: COALESCE Fallback Chain
```dax
column 'New Column Name' =
    COALESCE(
        LOOKUPVALUE('Primary Source'[Column], 'Primary Source'[Name], [Name]),
        LOOKUPVALUE('Fallback Source'[Column], 'Fallback Source'[Name], [Name]),
        BLANK()
    )
```

### Pattern: Boolean "In [System]" Flag
```dax
column 'In NewSystem' =
    IF(
        NOT(ISBLANK(LOOKUPVALUE('NewSystem'[Name], 'NewSystem'[Name], [Name]))),
        TRUE,
        FALSE
    )
```

### Pattern: CONCATENATEX for Multi-Value
```dax
column 'New Multi-Value' =
    VAR CurrentName = '_All Devices'[Name]
    VAR RelatedValues =
        CALCULATETABLE(
            VALUES('SourceTable'[ValueColumn]),
            'SourceTable'[NameColumn] = CurrentName
        )
    RETURN
        CONCATENATEX(
            DISTINCT(RelatedValues),
            'SourceTable'[ValueColumn],
            ", ",
            'SourceTable'[ValueColumn],
            ASC
        )
```

---

## Common Issues

### Credential Expiry
**Symptom**: Refresh fails with authentication errors
**Fix**: Go to File → Options → Data source settings and update credentials for the affected sources

### Source Schema Changes
**Symptom**: Refresh fails with "column not found" or similar errors
**Fix**: Open Power Query Editor, navigate to the affected query, and update column references to match the new schema. Check if column names or data types have changed in the source Excel files.

### Application Catalogue Path Change
**Symptom**: Application Catalogue queries fail
**Fix**: Update the `ApplicationCataloguePath` parameter in Power Query to point to the new file location

### Ops Theater Unavailable
**Symptom**: `Ops Theater RAETSMARINE Apps` query fails
**Fix**: Verify intranet access to `spdoca0001.raetsmarine.local`. If the server is permanently moved, update the URL in the expression. This is a web-scraped HTML table so it's fragile.

### New RVTools Export Format
**Symptom**: VMware queries fail after uploading new RVTools exports
**Fix**: Verify the new export has the same sheet names (`vInfo`, `vHost`, `vDisk`, `vCD`, `vNetwork`, `vSnapshot`, `vTools`) and column names. RVTools version upgrades may change column names.

### Diagnostics Queries Errors
**Symptom**: The 3 diagnostics expressions fail to load
**Fix**: These reference local file paths on a specific developer's machine (`C:\Users\GDVSHB\...`). They are developer-only diagnostic queries and can be safely ignored or deleted if they cause issues.

---

## Model Structure Files

The model is stored in TMDL (Tabular Model Definition Language) format:

```
True Reflection of Inventory (TRoI).SemanticModel\
  definition\
    model.tmdl              ← Model metadata, query groups, table refs
    relationships.tmdl      ← All 22 relationships
    expressions.tmdl        ← M functions, parameters, diagnostics
    tables\
      *.tmdl                ← One file per table (60 files)
    cultures\
      en-US.tmdl            ← Localisation

True Reflection of Inventory (TRoI).Report\
  report.json               ← Report pages, visuals, layout
  definition.pbir           ← Report definition
  .platform                 ← Platform metadata

True Reflection of Inventory (TRoI).SemanticModel\
  DAXQueries\
    _All Devices.dax        ← Development DAX query (may differ from live)
```

---

## Best Practices

1. **Always update the source Excel files before refreshing** — the model pulls snapshots, not live data
2. **Test changes in a copy** — the model is complex; breaking one calculated column can cascade
3. **Watch for duplicate Names** — the dedup logic in `_All Devices` relies on Name + VM uniqueness
4. **The Location column is fragile** — its deeply nested IF chain should be refactored if changes are needed (see [REVIEW] note in [05 — DAX Formulas](05-dax-formulas.md))
5. **Check the `.dax` file vs partition source** — the DAX query file may differ from the live model (e.g., SnowComputersFiltered is commented out in the `.dax` file but active in the partition)
