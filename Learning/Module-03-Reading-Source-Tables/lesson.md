# Module 03: Reading Source Tables

## What You Will Learn

How data gets into the TRoI model: Power Query (M) language, the anatomy of `.tmdl` files, SharePoint-based data sources, and the difference between imported and calculated columns.

**Prerequisites:** You should be comfortable with Power BI basics (Module 01) and understand the 61-table inventory landscape (Module 02).

---

## 1. Anatomy of a .tmdl File

Every table in the `tables/` folder is defined in a single `.tmdl` (Tabular Model Definition Language) file. These are plain-text files that Power BI reads to understand what each table looks like and where its data comes from.

Here is the actual `Active Directory Computers.tmdl` file, trimmed to show its key sections:

```
table 'Active Directory Computers'
    lineageTag: a17eed04-3c09-4d8a-a337-9a76fc2c9e6f

    column Name
        dataType: string
        lineageTag: 65502202-2279-4eb6-ae32-e846ddd28c13
        summarizeBy: none
        sourceColumn: Name

        annotation SummarizationSetBy = Automatic

    column Enabled
        dataType: boolean
        formatString: """TRUE"";""TRUE"";""FALSE"""
        lineageTag: ef5b0a59-6f01-4850-a0f9-bea45c65b300
        summarizeBy: none
        sourceColumn: Enabled

        annotation SummarizationSetBy = Automatic

    column Domain
        dataType: string
        lineageTag: 80ab2e62-5f6b-4e8b-8edd-93c5a04fe043
        summarizeBy: none
        sourceColumn: Domain

        annotation SummarizationSetBy = Automatic

    partition 'Active Directory Computers' = m
        mode: import
        queryGroup: 'Child Queries\Resource Management'
        source =
                let
                    Source = Excel.Workbook(Web.Contents("https://amlinonline.sharepoint.com/..."), null, true),
                    AllServers_Table = Source{[Item="AllServers",Kind="Table"]}[Data],
                    ...
                in
                    #"Renamed Columns"

    annotation PBI_NavigationStepName = Navigation
```

### What Each Section Means

**`table 'Active Directory Computers'`** -- The table definition header. The name in single quotes is how the table appears throughout the model and in DAX formulas.

**`column Name`** -- A column definition block. Each column has its own block with properties indented beneath it. The column name after `column` is how you reference it in DAX (e.g., `'Active Directory Computers'[Name]`).

**`dataType: string`** -- The data type of the column. Common types in this model include `string`, `int64`, `double`, `boolean`, and `dateTime`.

**`sourceColumn: Name`** -- This tells Power BI which column from the data source maps to this column definition. The presence of `sourceColumn` means this is an imported column (not calculated).

**`lineageTag: a17eed04-...`** -- A unique GUID used internally by Power BI for tracking. You can safely ignore these when reading `.tmdl` files.

**`summarizeBy: none`** -- The default aggregation behavior when the column is used in a visual. Text columns default to `none`. Numeric columns like `Logon Age` use `sum`.

**`annotation SummarizationSetBy = Automatic`** -- Power BI metadata. Annotations starting with `PBI_` or `SummarizationSetBy` are system-generated. You can ignore them when reading the file.

**`partition 'Active Directory Computers' = m`** -- The data source definition. This is where the Power Query (M) code lives. The `= m` suffix tells you this partition uses Power Query to import data. You will also encounter `= calculated`, which means the table is built by a DAX expression instead.

**`mode: import`** -- The data access mode. All tables in TRoI use `import` mode, meaning data is loaded as a snapshot during refresh rather than queried live from the source.

**`queryGroup: 'Child Queries\Resource Management'`** -- Indicates which folder group this query appears in within Power Query Editor. This is purely organizational.

---

## 2. Power Query (M) Basics

Power Query uses its own language called M. It is a functional, case-sensitive language designed for data extraction and transformation. Every partition with `= m` contains an M query.

### The let...in Structure

Every M query follows this pattern:

```
let
    Step1 = SomeFunction(...),
    Step2 = Transform(Step1),
    Step3 = AnotherTransform(Step2)
in
    Step3
```

Key rules:
- Each step is a named variable assignment.
- Steps are separated by commas.
- Steps build on previous steps -- `Step2` references `Step1`, `Step3` references `Step2`.
- The `in` clause specifies what gets returned. It is almost always the last step.
- Step names can contain spaces if wrapped in `#"..."` syntax, e.g., `#"Changed Type"`.

### Common M Functions in This Model

| Function | Purpose | Example |
|----------|---------|---------|
| `Web.Contents(url)` | Fetch content from a URL (SharePoint, web) | Connects to SharePoint file URLs |
| `Excel.Workbook(content)` | Open an Excel workbook from binary content | Reads the `.xlsx` file fetched by `Web.Contents` |
| `Web.BrowserContents(url)` | Fetch HTML from a web page | Used by `Ops Theater RAETSMARINE Apps` |
| `Table.TransformColumnTypes(table, types)` | Set column data types | Applied in nearly every query |
| `Table.NestedJoin(table1, key1, table2, key2, newCol, joinKind)` | Join two tables (like SQL JOIN) | Joins AD Computers with Domain mapping |
| `Table.ExpandTableColumn(table, col, names)` | Expand a nested/joined table column | Extracts columns after a join |
| `Table.PromoteHeaders(table)` | Use the first row as column headers | Used when reading Excel sheets (not named tables) |
| `Table.Distinct(table, columns)` | Remove duplicate rows | Deduplication after sorting |
| `Table.Combine(tables)` | Stack tables vertically (like SQL UNION) | Combines 3 vCenter exports in VMware VMs |
| `Table.SelectRows(table, condition)` | Filter rows based on a condition | Removes unwanted records |
| `Table.Sort(table, columns)` | Sort rows by one or more columns | Usually precedes `Table.Distinct` |
| `Table.Buffer(table)` | Materialize a table in memory | Ensures sort order is preserved before dedup |
| `Table.AddColumn(table, name, formula)` | Add a computed column | Creates new columns during transformation |
| `Table.RemoveColumns(table, columns)` | Drop columns from a table | Removes intermediate or unneeded columns |
| `Table.RenameColumns(table, renames)` | Rename columns | Cleans up column names |
| `Table.ReorderColumns(table, order)` | Rearrange column order | Controls final column layout |
| `Text.Upper(text)` | Convert text to uppercase | Normalizes server names for matching |

### The #"..." Syntax

In M, when a step name contains spaces or special characters, it must be wrapped in `#"..."`. This is why you see step names like:

```
#"Changed Type" = Table.TransformColumnTypes(...)
#"Uppercased Text" = Table.TransformColumns(...)
#"Merged DomainList = Domain on FQDN" = Table.NestedJoin(...)
```

These are just variable names. The `#"..."` wrapper is M's way of handling names that are not valid identifiers.

---

## 3. Real Example: Active Directory Computers

**File:** `tables/Active Directory Computers.tmdl`

This table imports computer objects from Active Directory via a SharePoint-hosted Excel export.

### The Full M Query (Annotated)

```
let
    // Step 1: Open the Excel workbook from SharePoint
    Source = Excel.Workbook(
        Web.Contents("https://amlinonline.sharepoint.com/.../MSA Report - All Servers.xlsx"),
        null, true),

    // Step 2: Navigate to the "AllServers" named table within the workbook
    AllServers_Table = Source{[Item="AllServers",Kind="Table"]}[Data],

    // Step 3: Set column data types (string, boolean, datetime, int64, etc.)
    #"Changed Type" = Table.TransformColumnTypes(AllServers_Table, {
        {"Name", type text}, {"Enabled", type logical}, {"Domain", type text},
        {"IP Address", type text}, {"Last Logon", type datetime},
        {"Logon Age", Int64.Type}, {"OS", type text}, ...
    }),

    // Step 4: Convert server names to uppercase for consistent matching
    #"Uppercased Text" = Table.TransformColumns(
        #"Changed Type", {{"Name", Text.Upper, type text}}),

    // Step 5: Join with domain mapping table to convert FQDN to NetBIOS domain name
    #"Merged DomainList = Domain on FQDN" = Table.NestedJoin(
        #"Uppercased Text", {"Domain"},
        #"Mapping DomainFQDN to DomainNetBIOS", {"FQDN"},
        "DomainList", JoinKind.LeftOuter),

    // Step 6: Extract the NetBIOS name from the joined table
    #"Expanded DomainList" = Table.ExpandTableColumn(
        #"Merged DomainList = Domain on FQDN", "DomainList",
        {"NetBIOS"}, {"DomainList.NetBIOS"}),

    // Step 7: Reorder columns for readability
    #"Reordered Columns" = Table.ReorderColumns(#"Expanded DomainList", {...}),

    // Step 8: Buffer and sort (sort by Name ascending, then Logon Age ascending)
    #"Sorted Rows" = Table.Buffer(
        Table.Sort(#"Reordered Columns", {
            {"Name", Order.Ascending}, {"Logon Age", Order.Ascending}
        })),

    // Step 9: Keep only the first row per server Name (deduplication)
    #"Removed Duplicates" = Table.Distinct(#"Sorted Rows", {"Name"}),

    // Step 10: Drop original FQDN Domain column, rename NetBIOS to "Domain"
    #"Removed Columns" = Table.RemoveColumns(#"Removed Duplicates", {"Domain"}),
    #"Renamed Columns" = Table.RenameColumns(
        #"Removed Columns", {{"DomainList.NetBIOS", "Domain"}})
in
    #"Renamed Columns"
```

### Key Columns

| Column | Data Type | Purpose |
|--------|-----------|---------|
| Name | string | Server name (uppercased for matching) |
| Enabled | boolean | Whether the AD computer account is enabled |
| Domain | string | NetBIOS domain name (after FQDN-to-NetBIOS mapping) |
| IP Address | string | IP address recorded in AD |
| Last Logon | dateTime | When the computer last authenticated |
| Logon Age | int64 | Number of days since last logon |
| OS | string | Operating system as recorded in AD |
| Description | string | AD description field |
| DN | string | Distinguished Name (full LDAP path) |
| Created | dateTime | When the AD object was created |
| Modified | dateTime | When the AD object was last modified |
| Password Last Set | dateTime | When the machine account password was last changed |

### How _All Devices Uses This Table

The `_All Devices` calculated table references Active Directory Computers in several LOOKUPVALUE chains:

```dax
// Domain lookup -- AD is first priority
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[Domain],
                'Active Directory Computers'[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Domain], 'Snow Computers'[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Domain], 'SolarWinds Nodes'[Name], [Name])
)

// "In AD" boolean flag
IF(
    NOT(ISBLANK(LOOKUPVALUE('Active Directory Computers'[Name],
                             'Active Directory Computers'[Name], [Name]))),
    TRUE,
    FALSE
)

// OS lookup -- AD is first priority in the fallback chain
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[OS],
                'Active Directory Computers'[Name], [Name]),
    LOOKUPVALUE('ConfigMgr All Devices'[Operating System], ...),
    LOOKUPVALUE('Snow Computers'[Operating System], ...),
    ...
)
```

Notice the pattern: `_All Devices` does not use model relationships to look up data from AD Computers. It uses `LOOKUPVALUE` with the server `Name` as the matching key.

---

## 4. Real Example: VMware VMs (Complex Multi-Source Query)

**File:** `tables/VMware VMs.tmdl`

This is one of the most complex M queries in the model. It combines data from three separate vCenter environments into a single table.

### The M Query Structure

```
let
    Sheet = "vInfo",

    // Source 1: On-premises vCenter (SVCENT0005)
    #"Source-SVCENT0005" = Excel.Workbook(
        Web.Contents("https://...RVTools_export_all_SVCENT0005.xlsx"), null, true),
    #"Sheet-SVCENT0005" = #"Source-SVCENT0005"{[Item=Sheet,Kind="Sheet"]}[Data],
    #"Table-SVCENT0005" = Table.PromoteHeaders(#"Sheet-SVCENT0005", ...),

    // Source 2: Azure VMware Solution UK South
    #"Source-AVSUKSouth" = Excel.Workbook(
        Web.Contents("https://...RVTools_export_all_AVSUKSouth.xlsx"), null, true),
    #"Sheet-AVSUKSouth" = #"Source-AVSUKSouth"{[Item=Sheet,Kind="Sheet"]}[Data],
    #"Table-AVSUKSouth" = Table.PromoteHeaders(#"Sheet-AVSUKSouth", ...),

    // Source 3: Azure VMware Solution UK West
    #"Source-AVSUKWest" = Excel.Workbook(
        Web.Contents("https://...RVTools_export_all_AVSUKWest.xlsx"), null, true),
    #"Sheet-AVSUKWest" = #"Source-AVSUKWest"{[Item=Sheet,Kind="Sheet"]}[Data],
    #"Table-AVSUKWest" = Table.PromoteHeaders(#"Sheet-AVSUKWest", ...),

    // Combine all three into one table
    CombinedTable = Table.Combine({
        #"Table-SVCENT0005", #"Table-AVSUKSouth", #"Table-AVSUKWest"
    }),

    // Set data types for 70+ columns
    #"Changed Type" = Table.TransformColumnTypes(CombinedTable, {...}),

    // Uppercase VM names for consistent matching
    #"Uppercased Text" = Table.TransformColumns(
        #"Changed Type", {{"VM", Text.Upper, type text}}),

    // Join with mapping table to resolve VM names to server names
    #"Merged Queries" = Table.NestedJoin(
        #"Uppercased Text", {"VM"},
        #"Mapping VM to ServerName", {"VM"},
        "Mapping VM to ServerName", JoinKind.LeftOuter),

    // Expand the Name column from the mapping
    #"Expanded Mapping-VMwareVM-ServerName" = Table.ExpandTableColumn(...),

    // Rename "Memory" to "Memory (MB)" for clarity
    #"Renamed Columns" = Table.RenameColumns(..., {{"Memory", "Memory (MB)"}}),

    // Create a simplified Network column from Network #1
    #"Added Network Column" = Table.AddColumn(#"Renamed Columns", "Network",
        each if Text.Contains([#"Network #1"], "DMZ") = true
            then [#"Network #1"]
            else Text.Select([#"Network #1"], {"0".."9", "A", "L", "N", "V"})),

    // Sort by Powerstate descending (powered-on first), then PowerOn date
    #"Sorted Rows" = Table.Buffer(
        Table.Sort(#"Added Network Column", {
            {"Powerstate", Order.Descending},
            {"PowerOn", Order.Descending}
        })),

    // Dedup by VM name -- keeps the first (powered-on, most recent) row
    #"Removed Duplicates" = Table.Distinct(#"Sorted Rows", {"VM"}),

    // Final column ordering
    #"Reordered Columns" = Table.ReorderColumns(#"Removed Duplicates", {...})
in
    #"Reordered Columns"
```

### What Makes This Query Complex

1. **Three separate data sources.** Each vCenter exports an RVTools Excel file to SharePoint. The query opens all three and reads the same `vInfo` sheet from each.

2. **`Table.Combine` for vertical stacking.** The three tables are combined into one using `Table.Combine`, which is the M equivalent of SQL `UNION ALL`.

3. **`Table.PromoteHeaders` instead of named tables.** The vCenter exports use raw Excel sheets (not named Excel tables), so the first row must be promoted to column headers.

4. **70+ columns with explicit type assignments.** The `Table.TransformColumnTypes` step assigns types to every column from the RVTools export.

5. **VM-to-ServerName mapping join.** Not every VM name matches the server name (e.g., a VM might be named `APP-SERVER-01-V` while the server name is `APP-SERVER-01`). The `Mapping VM to ServerName` table resolves these mismatches.

6. **Network name cleanup.** The `Added Network Column` step extracts a simplified network identifier from the full VLAN name.

7. **Sort-then-dedup pattern.** Rows are sorted so powered-on VMs come first, then deduplicated by VM name. This ensures that if a VM appears in multiple vCenters, the active instance is kept.

### Key Columns

This table has 70+ imported columns and 2 calculated columns. The most important ones for `_All Devices`:

| Column | Type | Notes |
|--------|------|-------|
| Name | string | Server name (from mapping join, may differ from VM name) |
| VM | string | VM name as it appears in vCenter |
| Host | string | ESXi host running the VM |
| Cluster | string | vSphere cluster |
| Datacenter | string | vSphere datacenter |
| Powerstate | string | "poweredOn", "poweredOff", "suspended" |
| CPUs | int64 | Virtual CPU count |
| Memory (MB) | int64 | Allocated memory in megabytes |
| Primary IP Address | string | Primary IP from VMware Tools |
| Network | string (added) | Simplified network identifier |
| Location | string (calculated) | DAX formula: derives location from Host and Datacenter |
| Memory | string (calculated) | DAX formula: formats memory as "X GB" or "X TB" |

### Calculated Columns on VMware VMs

This table has two DAX calculated columns defined directly on it (not on `_All Devices`):

```dax
// Location -- derives from Host name patterns and Datacenter values
column Location =
    SWITCH(
        TRUE(),
        CONTAINSSTRING([Host], "uksouth"), "AVS - UK South",
        CONTAINSSTRING([Host], "ukwest"), "AVS - UK West",
        [Datacenter] = "Paris Garden - HH3", "Hemel Hempstead",
        [Datacenter] = "Amlin-BranchOffices", "Leadenhall Building",
        [Datacenter]
    )

// Memory -- formats raw MB into human-readable GB/TB
column Memory =
    VAR MemoryMB = [Memory (MB)]
    VAR MemoryGB = MemoryMB / 1024
    VAR MemoryTB = MemoryMB / (1024 * 1024)
    RETURN
    SWITCH(
        TRUE(),
        MemoryTB >= 1, FORMAT(MemoryTB, "#,##0") & " TB",
        MemoryGB >= 1, FORMAT(MemoryGB, "#,##0") & " GB",
        FORMAT(MemoryMB, "#,##0") & " MB"
    )
```

These calculated columns appear in the `.tmdl` file with `= <DAX>` after the column name, as opposed to imported columns which have a `sourceColumn` property.

---

## 5. Real Example: Mapping Location (Embedded Data)

**File:** `tables/Mapping Location.tmdl`

Not all tables pull data from SharePoint. Some mapping tables embed their data directly inside the M query using compressed Base64-encoded JSON.

### The Full M Query

```
let
    Source = Table.FromRows(
        Json.Document(
            Binary.Decompress(
                Binary.FromText(
                    "fY+7DoMwDEV/xWKmEmqHzi0M...base64 string...",
                    BinaryEncoding.Base64),
                Compression.Deflate)),
        let _t = ((type nullable text) meta [Serialized.Text = true])
        in type table [#"Existing Location" = _t, #"Correct Location" = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source, {
        {"Existing Location", type text},
        {"Correct Location", type text}
    }),
    #"Sorted Rows" = Table.Sort(#"Changed Type", {
        {"Correct Location", Order.Ascending}
    })
in
    #"Sorted Rows"
```

### How the Embedded Data Pattern Works

Reading from the inside out:

1. `Binary.FromText("...", BinaryEncoding.Base64)` -- Decodes a Base64 string into raw binary data.
2. `Binary.Decompress(..., Compression.Deflate)` -- Decompresses the binary data using the Deflate algorithm.
3. `Json.Document(...)` -- Parses the decompressed binary as JSON, producing a list of row values.
4. `Table.FromRows(...)` -- Converts the list of rows into a table with the specified column schema.

The result is a simple two-column lookup table:

| Existing Location | Correct Location |
|-------------------|------------------|
| (raw/variant names from source systems) | (standardized location name) |

### Why Embed Data This Way?

- The mapping data is small and rarely changes.
- It avoids creating an external file dependency on SharePoint.
- It keeps the mapping self-contained within the model definition.
- Power BI Desktop generates this encoding automatically when you use the "Enter Data" feature.

Other mapping tables in this model use the same pattern, including `Mapping DomainFQDN to DomainNetBIOS` and `Mapping VM to ServerName`.

---

## 6. The SharePoint Pattern

The majority of the 61 tables in TRoI follow the same import pattern. Understanding this pattern lets you quickly read any source table in the model.

### Standard SharePoint Import Pattern

```
let
    // 1. Connect to the SharePoint-hosted Excel file
    Source = Excel.Workbook(
        Web.Contents("https://amlinonline.sharepoint.com/sites/.../filename.xlsx"),
        null, true),

    // 2. Navigate to the specific sheet or named table
    DataTable = Source{[Item="TableName", Kind="Table"]}[Data],
    //   -- OR for sheets without named tables --
    DataSheet = Source{[Item="SheetName", Kind="Sheet"]}[Data],
    Headers = Table.PromoteHeaders(DataSheet),

    // 3. Set column data types
    #"Changed Type" = Table.TransformColumnTypes(DataTable, {
        {"Column1", type text}, {"Column2", Int64.Type}, ...
    }),

    // 4. Apply transformations (filter, rename, join, dedup, etc.)
    #"Filtered Rows" = Table.SelectRows(#"Changed Type",
        each [Status] <> "Deleted"),

    // 5. Return the final result
    Result = #"Filtered Rows"
in
    Result
```

### Named Tables vs. Sheets

There is a subtle but important difference in Step 2:

- **Named Table:** `Source{[Item="AllServers", Kind="Table"]}[Data]` -- The Excel file contains a named table object. Column headers are already defined.
- **Sheet:** `Source{[Item="vInfo", Kind="Sheet"]}[Data]` -- The query reads a raw worksheet. The first row is data, not headers, so `Table.PromoteHeaders` is needed.

The Active Directory Computers query uses a named table (`AllServers`). The VMware VMs query uses raw sheets (`vInfo`) from RVTools exports.

### The ApplicationCataloguePath Parameter

Four tables in the model share the same SharePoint base URL for their source file. Instead of hardcoding the URL in each query, the model uses a parameter:

```
// Defined in expressions.tmdl:
expression ApplicationCataloguePath =
    "https://amlinonline.sharepoint.com/sites/.../Apps Cat updates....xlsx"
    meta [IsParameterQuery=true, Type="Any", IsParameterQueryRequired=true]
```

Any table that references `ApplicationCataloguePath` in its M query will use this URL. If the file moves to a different SharePoint location, you update the parameter once and all four tables pick up the change.

### Web Scraping (the Exception)

One table does not use SharePoint at all. `Ops Theater RAETSMARINE Apps` is defined in `expressions.tmdl` (not in `tables/`) and scrapes an internal HTML page:

```
let
    Source = Web.BrowserContents(
        "http://spdoca0001.raetsmarine.local/servers_func.html"),
    #"Extracted Table From Html" = Html.Table(Source, {
        {"Column1", "TABLE[id='servers_srt'] > * > TR > :nth-child(1)"},
        {"Column2", "TABLE[id='servers_srt'] > * > TR > :nth-child(2)"},
        ...
    }),
    ...
in
    #"Changed Type1"
```

This uses `Web.BrowserContents` to fetch HTML and `Html.Table` with CSS selectors to extract table data. It requires intranet access and will fail if the HTML structure changes. It is the only web-scraping source in the model.

---

## 7. Calculated vs. Imported Columns

When reading `.tmdl` files, you will encounter two types of columns. Knowing the difference is essential for understanding where data comes from.

### Imported Columns (from Power Query)

```
column Name
    dataType: string
    lineageTag: 65502202-...
    summarizeBy: none
    sourceColumn: Name
```

Identifying marks:
- Has a **`sourceColumn`** property -- the column name from the data source.
- Has an explicit **`dataType`** -- declared in the file.
- The data comes from Power Query at refresh time.

### Calculated Columns (DAX formula)

```
column Location =
    SWITCH(
        TRUE(),
        CONTAINSSTRING([Host], "uksouth"), "AVS - UK South",
        CONTAINSSTRING([Host], "ukwest"), "AVS - UK West",
        [Datacenter] = "Paris Garden - HH3", "Hemel Hempstead",
        [Datacenter] = "Amlin-BranchOffices", "Leadenhall Building",
        [Datacenter]
    )
    lineageTag: c182be05-...
    summarizeBy: none
```

Identifying marks:
- Has **`= <DAX expression>`** after the column name -- the formula that computes each row's value.
- **No `sourceColumn`** property -- the value is computed, not imported.
- **No explicit `dataType`** -- it is inferred from the formula result.
- The formula runs after data import, during model processing.

### Quick Reference

| Property | Imported Column | Calculated Column |
|----------|----------------|-------------------|
| `sourceColumn` | Present | Absent |
| `= <DAX>` after name | Absent | Present |
| `dataType` | Explicitly declared | Inferred from formula |
| Data origin | Power Query / M | DAX formula |
| When computed | During data refresh | After data refresh |

Most tables in TRoI contain only imported columns. The `_All Devices` calculated table, however, is built almost entirely from calculated columns using `LOOKUPVALUE` fallback chains. That topic is covered in detail in Module 08.

---

## 8. Expressions and Functions in expressions.tmdl

Beyond table definitions, the model also defines reusable M functions and parameters in `expressions.tmdl`. This file contains:

### Three M Functions

**`ExpandSubnet`** -- Takes a CIDR notation subnet (e.g., `10.0.0.0/24`) and returns a list of all IP addresses in that range. Used by network-related tables.

**`ConvertToMACAddress`** -- Converts a dot-separated MAC address format (e.g., `AABB.CCDD.EEFF`) to colon-separated format (`AA:BB:CC:DD:EE:FF`).

**`ConvertIPToDecimal`** -- Converts a dotted-decimal IP address (e.g., `10.0.0.1`) to its decimal integer representation (e.g., `167772161`). Used for IP range comparisons.

### One Parameter

**`ApplicationCataloguePath`** -- The SharePoint URL for the Application Catalogue Excel file, referenced by 4 tables (covered in Section 6 above).

### One Expression Table

**`Ops Theater RAETSMARINE Apps`** -- The web-scraped HTML table (covered in Section 6 above).

### Three Diagnostics Expressions

Three queries with names starting with `Qualys Assets_Source_` reference local file paths on a previous developer's machine (`C:\Users\GDVSHB\...`). These are Power Query diagnostics logs left behind from a debugging session. They will fail on any other machine and can be safely ignored.

---

## 9. Hands-On Exercise

1. Open the TRoI `.pbip` file in Power BI Desktop.
2. Open Power Query Editor: Home tab, click "Transform data."
3. In the Queries pane on the left, find **Active Directory Computers** under "Child Queries > Resource Management."
4. Click it. Look at the **Applied Steps** pane on the right side. Each step corresponds to a line in the `let...in` block you read in Section 3.
5. Click through each step one at a time. Watch the data preview change. Notice how:
   - `Source` shows the raw Excel workbook contents.
   - `AllServers_Table` narrows to the named table.
   - `Changed Type` assigns data types (column header icons change).
   - `Uppercased Text` converts the Name column to uppercase.
   - `Merged DomainList = Domain on FQDN` adds a nested table column.
   - `Expanded DomainList` flattens it into a regular column.
   - `Removed Duplicates` drops rows with duplicate server names.
6. Now find **VMware VMs** under "Child Queries > Hosting Platforms." Notice the three separate source steps and the `CombinedTable` step that stacks them.
7. Find **Mapping Location** under "Child Queries > Mapping." Click the `Source` step. The preview shows the decoded Base64 data as a normal table with two columns.
8. Close Power Query Editor (click "Close & Apply" or just close without applying).
9. Open `Active Directory Computers.tmdl` in a text editor. Match the file structure to what you just saw in Power BI:
   - Column definitions at the top correspond to the columns in the data preview.
   - The `partition` block contains the M code you stepped through.

---

## 10. Key Takeaways

- Every table in the model is defined in a `.tmdl` file containing column definitions and a data source (partition).
- Power Query (M) is the language used for data import and transformation. It uses a `let...in` structure where each step transforms the data incrementally.
- Most tables import from SharePoint-hosted Excel files using `Web.Contents` and `Excel.Workbook`.
- Some mapping tables embed small datasets directly as compressed Base64-encoded JSON inside the M query.
- One table (`Ops Theater RAETSMARINE Apps`) uses web scraping via `Web.BrowserContents`.
- The `partition '...' = m` marker means Power Query import. The `= calculated` marker means DAX (covered in Module 08).
- Imported columns have a `sourceColumn` property. Calculated columns have `= <DAX expression>` after the column name.
- The `ApplicationCataloguePath` parameter centralizes the SharePoint URL for four related tables.
- Reusable M functions (`ExpandSubnet`, `ConvertToMACAddress`, `ConvertIPToDecimal`) live in `expressions.tmdl`, not in individual table files.

---

**Next Module:** Module 04 covers how these individual source tables are organized into query groups and how the model's relationship structure connects them.
