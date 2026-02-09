# Module 13: Report Pages

## What You'll Learn
The 8 report pages, what each shows, how visuals connect to data.

### Sections:

**1. Report Structure Overview**
The report is defined in `True Reflection of Inventory (TRoI).Report\report.json` (~62K tokens -- very large). It contains 8 pages, each serving a specific analytical purpose. The pages use data from the 3 calculated tables and some source tables directly.

**2. Page-by-Page Walkthrough**

**Page 1: Server Inventory (Main Dashboard)**
The primary page most users interact with. Contains:
- KPI cards showing total device count, gap analysis numbers (Not In Qualys, Not In SEP, etc.)
- Slicers for filtering: Location, Device Type, Power State, Operating System, etc.
- A large table (tableEx visual) showing `_All Devices` data with columns like Name, Device Type, Location, Application, OS, etc.
- This is where users go to answer: "How many servers do we have? Where are the gaps?"

**Page 2: Azure vNet**
Focused on Azure virtual network mapping:
- Shows Azure VM assignments to virtual networks and subnets
- Uses the `Azure vNet` calculated column from `_All Devices`
- Helps network teams understand Azure network topology

**Page 3: Application Mapping Comparison**
Compares application names between the data model and the Application Catalogue:
- Shows matches and mismatches between `Application` and `Application (Application Catalogue)` columns
- Uses `Application Catalogue Discrepancies` and `Application Catalogue Missing From Data Model` tables
- Helps maintain data quality between systems

**Page 4: Asset Presence (WIP -- Work in Progress)**
Cross-system presence matrix:
- Shows which servers appear in which systems (AD, ConfigMgr, Snow, CMDB, etc.)
- Uses the boolean "In X" flags from `_All Devices`
- Visualizes the gap analysis in matrix form
- Marked as WIP -- may be incomplete or under development

**Page 5: Project Planning**
Migration planning dashboard using `_All Projects Scope`:
- Filtered view of servers to be migrated
- Groups by Wave, Application, Expected/Proposed DC Migration Path
- Uses the Applications, Servers, and Apps (Apps Cat) measures
- Primary tool for data center migration project managers

**Page 6: Citrix Published Applications**
Citrix application inventory:
- Uses `Citrix Team Published Applications` table
- Shows published Citrix applications and their server assignments
- Specific to the Citrix team's needs

**Page 7: Databases**
SQL database inventory:
- Uses `Merged SQL Databases`, `Azure Arc SQL Instances`, and related tables
- Shows database instances, servers, and database names
- Helps DBA teams track their SQL estate

**Page 8: Residual Servers Post AVS Migration**
Post-migration decommission tracking:
- Shows servers remaining in on-premises data centers after AVS migration
- Uses the `Has Matching AVS Server` column and Location filters
- Helps track which servers still need attention after migration waves

**3. Visual Types Used**
The report uses several visual types:
- **Card** -- Single KPI number (e.g., total device count, gap counts)
- **Slicer** -- Interactive filter controls (dropdowns, lists, search boxes)
- **Table (tableEx)** -- Data grids showing rows and columns
- **PowerSlicer** -- Custom visual for advanced multi-column filtering
- **textFilter** -- Custom visual for text-based search filtering
- Other standard visuals as needed

**4. How Slicers Work (Cross-Filtering)**
When you click a value in a slicer:
1. Power BI applies that filter to the underlying data table
2. All measures on the page recalculate under the new filter context
3. All other visuals on the page update to reflect the filtered data
4. Other slicers may update their available values (if cross-filtering is enabled)

Example: Select Location = "Romford" in the slicer -> the "No. of Devices" card shows only Romford servers -> the "Not In Qualys" card shows only Romford's Qualys gaps -> the table shows only Romford rows.

This is the power of the semantic model: define the data and relationships once, and the report automatically handles all the filtering and aggregation.

**5. Hands-On Exercise**
1. In Report view, click through all 8 page tabs at the bottom
2. On the Server Inventory page:
   a. Note the KPI cards at the top -- these are measures
   b. Click on a Location slicer value -- watch all visuals update
   c. Clear the filter (click the slicer eraser icon)
   d. Try filtering by Device Type
3. Navigate to Project Planning -- see the Wave and Migration Path data
4. Navigate to Databases -- see the SQL inventory
5. Navigate to Residual Servers Post AVS -- see the decommission tracking
6. On any page, try right-clicking a visual -> "Show as table" to see the underlying data

**6. Key Takeaways**
- 8 pages serve different audiences (operations, security, migration, DBA, Citrix)
- Server Inventory is the main dashboard -- gap analysis KPIs and full device table
- Slicers provide interactive filtering -- all visuals respond automatically
- The report is the "presentation layer" -- it reads from the semantic model
- `report.json` is very large (~62K tokens) -- target specific pages when investigating
