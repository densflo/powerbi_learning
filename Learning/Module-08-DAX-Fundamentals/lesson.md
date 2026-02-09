# Module 08: DAX Fundamentals

## What You'll Learn
DAX basics needed to understand this model -- functions, patterns, syntax.

### Sections:

**1. What is DAX?**
DAX (Data Analysis Expressions) is the formula language used in Power BI for:
- Calculated tables (entire tables built from formulas)
- Calculated columns (new columns added to existing tables via formulas)
- Measures (dynamic aggregations that respond to filter context)

DAX vs Power Query (M):
- **M** runs when data is imported/refreshed -- it defines HOW data gets into the model
- **DAX** runs after import -- it defines calculations ON TOP of the imported data
- Think of M as "ETL" and DAX as "analytics"

**2. Three DAX Object Types**

**Calculated Tables** -- Entire tables defined by a DAX expression:
```
partition '_All Devices' = calculated
    source =
        VAR AllDevices = UNION(...)
        ...
        RETURN result
```
In .tmdl files, these have `partition '...' = calculated` (vs `= m` for Power Query).

**Calculated Columns** -- New columns added to a table, computed row by row:
```
column 'In AD' =
    IF(
        NOT(ISBLANK(LOOKUPVALUE('Active Directory Computers'[Name], 'Active Directory Computers'[Name], [Name]))),
        TRUE,
        FALSE
    )
```
Calculated columns are evaluated once per row and stored in the model. They have access to "row context" -- they know which row they're computing for.

**Measures** -- Dynamic calculations evaluated at query time:
```
measure 'No. of Devices' = COUNTA([Name])
```
Measures respond to "filter context" -- their value changes based on slicers, filters, and visual groupings. They are NOT stored row by row; they're computed on the fly.

**3. Row Context vs Filter Context**
This is the most important DAX concept:

- **Row context**: When a calculated column formula runs, it knows the current row. `[Name]` refers to the Name value of THAT specific row. Each row computes independently.
- **Filter context**: When a measure evaluates, it considers all active filters (slicers, visual filters, row/column groupings). `COUNTA([Name])` counts all Name values visible under the current filters.

Example: If a slicer is set to Location = "Romford":
- A calculated column `In AD` still shows TRUE/FALSE per row (row context doesn't change)
- A measure `No. of Devices` shows only the count of Romford devices (filter context changes)

**4. Key Functions Used in This Model**

**LOOKUPVALUE(result_column, search_column, search_value)**
The workhorse of this model. Like VLOOKUP in Excel.
```dax
LOOKUPVALUE('Active Directory Computers'[Domain], 'Active Directory Computers'[Name], [Name])
```
Translation: "In the Active Directory Computers table, find the row where Name matches the current row's Name, and return the Domain value."
Returns BLANK if no match found. Errors if multiple matches found.

**COALESCE(value1, value2, value3, ...)**
Returns the first non-BLANK value. Used for fallback chains:
```dax
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[Domain], ...),
    LOOKUPVALUE('Snow Computers'[Domain], ...),
    LOOKUPVALUE('SolarWinds Nodes'[Domain], ...)
)
```
Translation: "Try AD first. If blank, try Snow. If blank, try SolarWinds."

**UNION(table1, table2, ...)**
Stacks tables vertically (all must have the same columns):
```dax
UNION(
    SELECTCOLUMNS('VMware VMs', "Name", [Name], "VM", [VM]),
    SELECTCOLUMNS('Azure VMs', "Name", [Name], "VM", BLANK())
)
```

**EXCEPT(table1, table2)**
Returns rows in table1 that are NOT in table2 (set subtraction):
```dax
EXCEPT(
    SELECTCOLUMNS(FILTER('SolarWinds VMs', ...), "Name", [Name], "VM", BLANK()),
    SELECTCOLUMNS(AllDevices, "Name", [Name], "VM", BLANK())
)
```
Translation: "Give me SolarWinds VMs that don't already appear in AllDevices."

**SELECTCOLUMNS(table, "NewName1", expression1, ...)**
Creates a new table with specified columns (schema standardization):
```dax
SELECTCOLUMNS('VMware VMs', "Name", 'VMware VMs'[Name], "VM", 'VMware VMs'[VM])
```
Essential for UNION -- all tables must have identical column names and count.

**FILTER(table, condition)**
Returns a filtered subset of rows:
```dax
FILTER('SolarWinds VMs', 'SolarWinds VMs'[Virtualization Platform] = "Hyper-V")
```

**CALCULATE(expression, filter1, filter2, ...)**
Evaluates an expression under modified filter context:
```dax
CALCULATE(
    COUNTA('_All Devices'[In Qualys]),
    '_All Devices'[In Qualys] IN { FALSE }
)
```
Translation: "Count In Qualys values, but only where In Qualys = FALSE."

**CONCATENATEX(table, expression, delimiter)**
Concatenates values from multiple rows into a single string:
```dax
CONCATENATEX(
    DISTINCT(RelatedValues),
    'Mapping Server to Application'[Application],
    ", ",
    'Mapping Server to Application'[Application],
    ASC
)
```
Translation: "Join all application names with commas, sorted alphabetically."

**GENERATE(table1, table2_expression)**
Creates a Cartesian product (cross join) -- for each row in table1, evaluate table2:
```dax
GENERATE(
    '_All Devices',
    FILTER('Mapping Server To Application', [Server] = '_All Devices'[Name])
)
```
Used by `_All Projects Scope` and `_All Applications by Server`.

**IF() / SWITCH(TRUE(), ...)**
Conditional logic:
```dax
IF(condition, true_value, false_value)

SWITCH(
    TRUE(),
    condition1, result1,
    condition2, result2,
    default_result
)
```
SWITCH(TRUE(), ...) is like an if/else if chain -- evaluates conditions top to bottom, returns the first TRUE match.

**ISBLANK() / NOT()**
Null handling:
```dax
IF(NOT(ISBLANK(LOOKUPVALUE(...))), TRUE, FALSE)
```
Translation: "If the lookup returns a value (not blank), then TRUE."

**CONTAINS(table, column, value) / CONTAINSSTRING(text, substring)**
- CONTAINS: Checks if a value exists in a table column (returns TRUE/FALSE)
- CONTAINSSTRING: Checks if a substring exists within a text string

**5. How to Read DAX in a .tmdl File**
In `.tmdl` files, DAX appears in two places:

Column definitions (calculated columns):
```
column 'Column Name' =
    <DAX expression>
    lineageTag: ...
```
Or for multiline:
```
column 'Column Name' = ```
    <multiline DAX>
    ```
    lineageTag: ...
```

Partition sources (calculated tables):
```
partition 'Table Name' = calculated
    mode: import
    source =
        <DAX expression>
```
Or multiline with triple backticks.

**6. The VAR...RETURN Pattern**
Complex DAX uses variables for readability:
```dax
VAR ServerName = [Name]
VAR VMName = [VM]
VAR VMwareTotalMiB = CALCULATE(SUM('VMware Disks'[Capacity MiB]), ...)
VAR AzureTotalGiB = CALCULATE(SUM('Azure Disks'[diskSizeGB]), ...)
VAR TotalMiB = VMwareTotalMiB + (AzureTotalGiB * 1024)
RETURN
    IF(TotalMiB > 0, FORMAT(TotalMiB, "#,##0"), BLANK())
```
VARs are evaluated once and can be referenced multiple times. The RETURN clause specifies the final result.

**7. Hands-On Exercise**
1. In Power BI Desktop, go to Data view
2. Click on `_All Devices` table
3. Right-click on the `Domain` column header or select it and check the formula bar -- see the COALESCE expression
4. Right-click on the `In AD` column -- see the LOOKUPVALUE pattern
5. Click on the `No. of Devices` measure in the Fields pane -- see the formula bar show `COUNTA([Name])`
6. Open `_All Devices.tmdl` in a text editor -- find these same formulas in the file

**8. Key Takeaways**
- DAX handles calculated tables, calculated columns, and measures
- LOOKUPVALUE is the model's workhorse (like VLOOKUP)
- COALESCE creates fallback chains across multiple source tables
- UNION + EXCEPT handle set operations for deduplication
- Row context (columns) vs filter context (measures) is the key distinction
- VAR...RETURN makes complex DAX readable
