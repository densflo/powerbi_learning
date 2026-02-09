# Module 14: Maintenance & Gotchas

## What You'll Learn
How to maintain and extend the model, known issues, and reference material.

### Sections:

**1. Refreshing Data**
To update the model with current data:
1. Open the .pbip in Power BI Desktop
2. Click **Home -> Refresh** (or Ctrl+Alt+F5 for full refresh)
3. Power Query will re-execute all M queries, fetching fresh data from SharePoint
4. DAX calculated tables will recalculate based on the new imported data
5. Report visuals will reflect the updated data

Common refresh issues:
- SharePoint credentials may expire -- re-enter when prompted
- Network connectivity required for SharePoint sources
- Ops Theater web scrape requires intranet access
- Large refreshes may take several minutes

**2. How to Add a New Source System**
If a new IT tool needs to be integrated, follow this pattern:

**Step 1: Import the data**
- Create a new Power Query (M) query to import the data (usually from SharePoint Excel)
- Add the query to the appropriate query group in Power Query Editor
- This creates a new `.tmdl` file in the `tables/` folder

**Step 2: (Optional) Add to _All Devices UNION**
If the new system discovers servers not already tracked:
- Add a new VAR in the _All Devices partition source
- Use FILTER to select relevant rows
- Use SELECTCOLUMNS to standardize to Name + VM
- Use EXCEPT against AllDevices to deduplicate
- Add the new VAR to the CombinedDevices UNION

**Step 3: Add a boolean flag column**
In _All Devices, add a calculated column:
```dax
column 'In NewSystem' =
    IF(
        NOT(ISBLANK(LOOKUPVALUE('New System Table'[Name], 'New System Table'[Name], [Name]))),
        TRUE,
        FALSE
    )
```

**Step 4: Add a gap measure**
```dax
measure 'Not In NewSystem' =
    VAR NotInNewSystemCount =
    CALCULATE(
        COUNTA('_All Devices'[In NewSystem]),
        '_All Devices'[In NewSystem] IN { FALSE }
    )
    RETURN
    IF(NotInNewSystemCount = BLANK(), 0, NotInNewSystemCount)
```

**Step 5: Update report visuals**
- Add the new measure to KPI cards on the Server Inventory page
- Add the new boolean column as a slicer if needed

**3. How to Add a Calculated Column to _All Devices**
Common patterns for new columns:

**Single value lookup:**
```dax
column 'New Column' = LOOKUPVALUE('Source Table'[Column], 'Source Table'[Name], [Name])
```

**Fallback chain:**
```dax
column 'New Column' =
    COALESCE(
        LOOKUPVALUE('Table A'[Column], 'Table A'[Name], [Name]),
        LOOKUPVALUE('Table B'[Column], 'Table B'[Name], [Name])
    )
```

**Multi-value concatenation:**
```dax
column 'New Column' =
    VAR CurrentName = [Name]
    VAR RelatedValues = CALCULATETABLE(VALUES('Source'[Value]), 'Source'[Name] = CurrentName)
    RETURN CONCATENATEX(DISTINCT(RelatedValues), [Value], ", ", [Value], ASC)
```

**Boolean flag:**
```dax
column 'Has Something' =
    IF(NOT(ISBLANK(LOOKUPVALUE('Source'[Name], 'Source'[Name], [Name]))), TRUE, FALSE)
```

**4. Known Gotchas**

**4.1 .dax File vs Live Partition**
The file `DAXQueries\_All Devices.dax` is a development workspace. It comments out `SnowComputersFiltered` in the UNION. The live model (in `_All Devices.tmdl`) includes it. Always trust the `.tmdl` file as the source of truth. If you edit DAX in the .dax file, you must also update the partition source in the .tmdl.

**4.2 Diagnostics Expressions**
Three queries in `expressions.tmdl` reference `C:\Users\GDVSHB\...` -- a previous developer's machine. These are:
- `Qualys Assets_Source_Detailed_2025-09-19_17:11`
- `Qualys Assets_Source_Aggregated_2025-09-19_17:11`
- `Qualys Assets_Source_Partitions_2025-09-19_17:11`

They will fail on any other machine. They are in the "Diagnostics" query group and can be safely ignored or removed. They were one-time performance diagnostics.

**4.3 Ops Theater Web Scraping**
`Ops Theater RAETSMARINE Apps` scrapes `http://spdoca0001.raetsmarine.local/servers_func.html`. This:
- Requires intranet access (will not work remotely without VPN)
- Is fragile -- any HTML structure change will break the import
- Uses CSS selectors targeting `TABLE[id='servers_srt']`
- If it breaks, the table will fail to refresh (other tables will still refresh)

**4.4 Location Column Complexity**
The Location column in `_All Devices` is ~140 lines of deeply nested IF/LOOKUPVALUE. The same 5-tier fallback logic is duplicated across 3 branches (VMware, Azure, other). This is:
- Functional but fragile to modify
- Easy to introduce bugs if editing one branch but not others
- A candidate for refactoring (but works reliably as-is)

**4.5 ApplicationCataloguePath Parameter**
The parameter `ApplicationCataloguePath` controls the SharePoint URL for 4 Application Catalogue tables. If the Excel file moves to a new SharePoint location:
1. Update the parameter value in expressions.tmdl or in Power BI (Home -> Transform Data -> Manage Parameters)
2. All 4 tables will automatically point to the new location

**4.6 The VM vs Name Column**
Throughout the model, VM refers to the VMware display name (often FQDN) and Name is the short hostname. Mixing these up in LOOKUPVALUE calls can cause blank results. Always check which column the source table uses.

**5. The VMware VMs Check Secondary Model**
The repository also contains `VMware VMs Check.pbip` -- a separate, simpler model:
- 6 tables, 4 relationships, 1 empty report page
- Used for ad-hoc VMware VM verification
- Independent from the main TRoI model

**6. Existing Documentation**
The `docs/` folder contains generated documentation files (01-09):
- These were created to document the model's architecture
- Use them as supplementary reference material
- `docs/09-maintenance-guide.md` is particularly useful for ongoing maintenance patterns

**7. Summary -- Your TRoI Knowledge Map**
After completing all 14 modules, you understand:
- **Module 01-02**: Power BI fundamentals and the 61-table model map
- **Module 03-06**: How data gets in (Power Query, source tables, mapping tables)
- **Module 07**: How tables connect (22 relationships, 5 clusters)
- **Module 08**: DAX language fundamentals
- **Module 09-11**: The _All Devices engine (UNION chain, columns, measures)
- **Module 12**: Project planning tables (GENERATE-based)
- **Module 13**: Report pages and interactive visuals
- **Module 14**: Maintenance, extension, and known issues

You are now equipped to:
- Navigate and understand any part of the TRoI model
- Trace data flow from source to report
- Add new source systems following established patterns
- Troubleshoot common issues
- Maintain and extend the model with confidence

**8. Hands-On Exercise**
1. Read `docs/09-maintenance-guide.md` in the docs folder
2. Practice an end-to-end trace: pick a server name visible in the Server Inventory report
   a. Find it in `_All Devices` (Data view)
   b. Check its Device Type -- which source did it come from?
   c. Find the original row in that source table (e.g., VMware VMs)
   d. Check its boolean flags -- which systems track this server?
   e. Check its Location -- which fallback tier provided the value?
3. Review the `expressions.tmdl` file -- identify the diagnostics queries
4. Open the `ApplicationCataloguePath` parameter -- see its current value

**9. Key Takeaways**
- Adding new sources follows a 5-step pattern: Import -> UNION -> Flag -> Measure -> Report
- The .dax file is for development; the .tmdl is the source of truth
- Three diagnostics expressions reference a previous developer's path -- ignore them
- Ops Theater web scraping and the Location column are the most fragile components
- Keep the `ApplicationCataloguePath` parameter updated when the source file moves
- Use the docs/ folder and this course as ongoing reference material
