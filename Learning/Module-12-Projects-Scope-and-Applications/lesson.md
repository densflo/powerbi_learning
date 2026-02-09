# Module 12: Projects Scope & Applications

## What You'll Learn
The other 2 calculated tables that build on `_All Devices` -- `_All Projects Scope` and `_All Applications by Server`.

### Sections:

**1. How GENERATE Works**
Both tables use the GENERATE function to create a cross-product:
```dax
GENERATE(
    '_All Devices',
    FILTER('Mapping Server To Application', condition)
)
```
GENERATE takes each row from `_All Devices` and, for that row, evaluates the FILTER expression on `Mapping Server to Application`. If the server maps to 3 applications, GENERATE produces 3 rows (one per application). If it maps to zero applications, the server is excluded entirely.

Think of it like a SQL CROSS APPLY or INNER JOIN -- it multiplies rows based on the mapping.

**2. _All Projects Scope -- Migration Planning**

**Purpose**: Shows which servers need to migrate (for data center migration projects), broken down by application. Excludes servers already in AVS/New ALZ and "Client Endpoint" applications.

**Partition source (actual DAX):**
```dax
DISTINCT(
    SELECTCOLUMNS(
        GENERATE(
            '_All Devices',
            FILTER(
                'Mapping Server To Application',
                'Mapping Server To Application'[Server] = '_All Devices'[Name] &&
                'Mapping Server To Application'[Application] <> "Client Endpoint" &&
                NOT CONTAINSSTRING('_All Devices'[Location], "AVS") &&
                NOT CONTAINSSTRING('_All Devices'[Location], "New ALZ")
            )
        ),
        "Name", '_All Devices'[Name],
        "VM Name", '_All Devices'[VM],
        "Device Type", '_All Devices'[Device Type],
        "Power State", '_All Devices'[Power State],
        "Location", '_All Devices'[Location],
        "CMDB Status", '_All Devices'[CMDB Status],
        "Decom CHG", '_All Devices'[Decom CHG Number],
        "Application", 'Mapping Server To Application'[Application],
        "IP Address", '_All Devices'[IP Address],
        "Environment", '_All Devices'[Environment]
    )
)
```

Key filters:
- `Application <> "Client Endpoint"` -- Excludes client endpoints (not servers)
- `NOT CONTAINSSTRING(Location, "AVS")` -- Excludes servers already in Azure VMware Solution
- `NOT CONTAINSSTRING(Location, "New ALZ")` -- Excludes servers already in New Azure Landing Zone

SELECTCOLUMNS picks specific columns from the cross-product. DISTINCT removes any duplicate rows.

**Calculated columns (11):**
- `Application (Application Catalogue)` -- Maps application name to catalogue name
- `Application Reference (Application Catalogue)` -- Catalogue reference ID
- `Application Name Match` -- TRUE/FALSE if app name exactly matches catalogue
- `Wave` -- Migration wave from Application Catalogue Wave Plan
- `Expected DC Migration Path` -- Expected data center handling
- `Proposed DC Migration Path` -- Proposed data center handling
- `KS Comments` -- Comments from the mapping table
- `Network` -- VLAN/network lookup (same pattern as _All Devices)
- `Domain` -- COALESCE fallback (AD -> Snow -> SolarWinds)
- `Legal Entity Usage (Application Catalogue)` -- Legal entity from catalogue
- `Type` -- Maps "Business Application"->"Business", "Technology Component"->"Technology"

**Measures (3):**
```dax
measure Applications = DISTINCTCOUNTNOBLANK([Application])
measure Servers = DISTINCTCOUNTNOBLANK([Name])
measure 'Apps (Apps Cat)' = DISTINCTCOUNTNOBLANK([Application (Application Catalogue)])
```
Each wrapped in IF(BLANK(), 0, result) defensive pattern.

**3. _All Applications by Server -- Broader View**

**Purpose**: Similar to Projects Scope but WITHOUT the location filter -- includes all locations (AVS, New ALZ, everything). Adds an End of Life flag.

Key differences from Projects Scope:
| Aspect | _All Projects Scope | _All Applications by Server |
|--------|--------------------|-----------------------------|
| Location filter | Excludes AVS and New ALZ | Includes ALL locations |
| Client Endpoint | Excluded | Excluded (same) |
| End of Life flag | No | Yes (Windows 2008/2012) |
| Type mapping | Business/Technology from catalogue | May differ |
| Purpose | Migration planning | Application-to-server mapping across all locations |

The End of Life column checks if the Operating System contains "2008" or "2012" (Windows Server versions past end of life).

**4. Comparing the Two Tables Side by Side**
Both are GENERATE-based cross-products of `_All Devices` x `Mapping Server to Application`. The key difference is the FILTER conditions:
- Projects Scope: Narrower (excludes migrated servers) -- for active migration planning
- Applications by Server: Broader (includes everything) -- for comprehensive application inventory

**5. Hands-On Exercise**
1. In Data view, click on `_All Projects Scope` -- note the row count
2. Click on `_All Applications by Server` -- note the higher row count (no location filter)
3. In `_All Projects Scope`, look at the Application column -- find servers with multiple rows (same Name, different Application)
4. Check the Wave column -- see migration wave assignments
5. Filter to a specific Application -- see all servers running that application
6. Open `_All Projects Scope.tmdl` in a text editor -- find the GENERATE expression

**6. Key Takeaways**
- GENERATE creates a cross-product: each server x each application = one row
- `_All Projects Scope` filters to migration-relevant servers (excludes AVS/ALZ)
- `_All Applications by Server` includes all locations for complete mapping
- Both build on `_All Devices` -- same foundation, different views
- 11 calculated columns add application catalogue data and migration planning info
- 3 measures provide distinct counts of applications, servers, and catalogue apps
