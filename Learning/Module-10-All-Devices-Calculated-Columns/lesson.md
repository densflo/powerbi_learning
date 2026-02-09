# Module 10: _All Devices -- Calculated Columns

## What You'll Learn
The 30+ calculated columns that enrich each device row with data from across all source systems.

### Sections:

**1. How Calculated Columns Work in _All Devices**
After the UNION chain produces rows with just Name and VM, Power BI evaluates 30+ calculated column formulas for every row. Each column reaches into source tables using LOOKUPVALUE, COALESCE, CONTAINS, or more complex logic.

Remember: `_All Devices` has NO relationships. Every cross-table lookup is explicit in DAX.

**2. Boolean Flags -- "Is this server in System X?" (9 columns)**
These all follow the same pattern:
```dax
column 'In AD' =
    IF(
        NOT(ISBLANK(LOOKUPVALUE('Active Directory Computers'[Name], 'Active Directory Computers'[Name], [Name]))),
        TRUE,
        FALSE
    )
```
Translation: "Look up the current server's Name in AD Computers. If found (not blank), TRUE. Otherwise FALSE."

All 9 boolean flag columns:
| Column | Source Table | Purpose |
|--------|------------|---------|
| In AD | Active Directory Computers | Is the server in Active Directory? |
| In CM | ConfigMgr All Devices | Is it managed by ConfigMgr/SCCM? |
| In CMDB | ServiceNow CMDB | Is it registered in the CMDB? |
| In Snow | Snow Computers | Is it tracked by Snow Software? |
| In Qualys | Qualys Assets | Is it scanned by Qualys? |
| In SEP | Symantec Endpoint Protection Devices | Does it have antivirus? |
| In Panaseer | Panaseer All Devices | Is it in Panaseer? |
| In SolarWinds Nodes | SolarWinds Nodes | Is it monitored by SolarWinds? |
| In SolarWinds VMs | SolarWinds VMs | Is it discovered as a SolarWinds VM? |

These flags drive the gap analysis measures (Module 11).

**3. Boolean Characteristics -- Server Properties (5 columns)**
Similar boolean logic but checking for specific characteristics:

| Column | Logic | Purpose |
|--------|-------|---------|
| Encrypted Drives | CONTAINS('ConfigMgr Encryption Status', Name) | Has BitLocker? |
| Snapshots | CONTAINS('VMware Snapshots', Name) | Has VMware snapshots? |
| Mounted Media | CONTAINS('VMware CD', VM) | Has mounted CD/DVD? |
| E1000 Adapter | COUNTROWS(FILTER('VMware Network', VM match AND Adapter <> "Vmxnet3")) > 0 | Has legacy network adapter? |
| Has Matching AVS Server | Complex check for matching server in AVS locations | Is being migrated to AVS? |

Note: E1000 Adapter uses FILTER + COUNTROWS instead of CONTAINS because it needs to check a condition (Adapter <> "Vmxnet3"), not just existence.

**4. COALESCE Fallback Chains -- "Try source A, then B, then C" (6 columns)**
These try multiple source tables in priority order, returning the first non-blank value:

**Domain:**
```dax
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[Domain], 'Active Directory Computers'[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Domain], 'Snow Computers'[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Domain], 'SolarWinds Nodes'[Name], [Name])
)
```
Priority: AD then Snow then SolarWinds

**Operating System:**
```dax
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[OS], ...),
    LOOKUPVALUE('ConfigMgr All Devices'[Operating System], ...),
    LOOKUPVALUE('Snow Computers'[Operating System], ...),
    LOOKUPVALUE('SolarWinds Nodes'[Machine Type], ...)
)
```
Priority: AD then ConfigMgr then Snow then SolarWinds

**Manufacturer:**
```dax
COALESCE(
    LOOKUPVALUE('Snow Computers'[Manufacturer], ...),
    LOOKUPVALUE('ConfigMgr All Devices'[Manufacturer], ...),
    LOOKUPVALUE('SolarWinds Nodes'[Manufacturer], ...)
)
```
Priority: Snow then ConfigMgr then SolarWinds

**Model:** Same priority as Manufacturer: Snow then ConfigMgr then SolarWinds

**Cores:**
```dax
COALESCE(
    LOOKUPVALUE('VMware VMs'[CPUs], 'VMware VMs'[VM], [VM]),
    LOOKUPVALUE('Azure VMs'[Cores], 'Azure VMs'[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Processor cores], 'Snow Computers'[Name], [Name])
)
```
Priority: VMware then Azure then Snow. Note: VMware uses VM column, others use Name.

**Memory:** Same priority as Cores: VMware then Azure then Snow

**5. Complex Logic Columns (10+ columns)**

**Device Type** -- Uses SWITCH(TRUE(), ...):
```dax
SWITCH(
    TRUE(),
    NOT(ISBLANK(LOOKUPVALUE('VMware VMs'[Name], ...))), "VMware",
    NOT(ISBLANK(LOOKUPVALUE('Azure VMs'[Name], ...))), "Azure",
    NOT(ISBLANK(CALCULATE(SELECTEDVALUE('SolarWinds VMs'[Name]), Hyper-V filter))), "Hyper-V",
    [Manufacturer] IN {"Dell Inc.", "HP", "HPE", "IBM"}, "Physical",
    BLANK()
)
```
Checks each source in priority order. Falls back to checking Manufacturer for physical servers.

**Application** -- Uses CONCATENATEX for many-to-many:
```dax
VAR RelatedValues = CALCULATETABLE(VALUES('Mapping Server to Application'[Application]), ...)
VAR SortedUniqueValues = CONCATENATEX(DISTINCT(RelatedValues), ..., ", ", ..., ASC)
RETURN SortedUniqueValues
```
Joins all application names for this server with commas, sorted alphabetically.

**IP Address** -- Priority fallback with filtering:
Tries Azure IP first, then VMware IPs (concatenated), then Merged IPs (filtering out 192.x.x.x and 169.x.x.x private/APIPA ranges).

**Location** -- The most complex column (~140 lines of nested IF/LOOKUPVALUE):
5-tier fallback duplicated across 3 branches (VMware, Azure, other):
1. VMware VMs Location (from vCenter)
2. Azure VMs Location
3. VMware Hosts Location (for ESXi hosts)
4. Merged Subnets Location (via IP Subnet)
5. Mapping Manual Server To Location (manual override)
The nesting is deep because each branch checks Device Type first, then falls through the location sources.

**Power State** -- SWITCH by Device Type:
- Physical: returns BLANK
- VMware: Translate "poweredOn" to "ON", "poweredOff" to "OFF"
- Azure: Translate "VM running" to "ON", "VM deallocated" to "OFF"

**Disk Capacity** -- Sums VMware disk MiB + Azure disk GiB, formats as MB/GB/TB.

**Environment** -- Looks up ServiceNow CMDB Environment using FIRSTNONBLANK + FILTER pattern.

**CMDB Status** -- CONCATENATEX of all Install Status values from ServiceNow CMDB (multiple records per server possible).

**Decom CHG Number** -- Searches ServiceNow Decommission Change Requests Description field for the server name using CONTAINSSTRING, returns concatenated CHG numbers.

**6. Simple Lookup Columns (5+ columns)**
Single LOOKUPVALUE with no fallback:

| Column | Expression |
|--------|-----------|
| Host | LOOKUPVALUE('VMware VMs'[Host], VM, [VM]) |
| Cluster | LOOKUPVALUE('VMware VMs'[Cluster], VM, [VM]) |
| VMware Tools Version | LOOKUPVALUE('VMware Tools'[Tools Version], VM, [VM]) |
| SEP Status | LOOKUPVALUE('Symantec Endpoint Protection Devices'[Security Status], Name, [Name]) |
| Owner | LOOKUPVALUE('ServiceNow CMDB'[Owned by], Name, [Name]) |
| Application (CMDB) | LOOKUPVALUE('ServiceNow CMDB'[Application name], Name, [Name]) |
| Operating System (VMware Tools) | LOOKUPVALUE('VMware VMs'[OS according to VMware Tools], VM, [VM]) |
| IP Subnet | LOOKUPVALUE('Merged Subnets Expanded'[Subnet], IP Address, [IP Address]) |
| Azure vNet | CALCULATETABLE + CONCATENATEX on Azure VirtualNetworks via subnetAddressPrefix |

**7. Additional Multi-Value Columns**
| Column | Pattern |
|--------|---------|
| Application (Environment Management Database) | CONCATENATEX of Environments Team data |
| IP Addresses | CONCATENATEX of all Merged IP Addresses (filtered to non-blank) |

**8. Hands-On Exercise**
1. Open `_All Devices.tmdl` in a text editor
2. Find the `column Domain =` definition -- trace the COALESCE to AD, Snow, SolarWinds
3. Find `column 'In AD' =` -- see the boolean flag pattern
4. Find `column 'Device Type' =` -- read the SWITCH(TRUE(), ...) logic
5. Find `column Location =` -- scroll through the ~140 lines of nested IF logic
6. In Power BI Data view, click on `_All Devices`, find these columns, and examine their values
7. Pick a server name, then check if its Domain value matches what you see in Active Directory Computers

**9. Key Takeaways**
- 30+ calculated columns enrich the raw Name/VM foundation
- 9 boolean flags check "Is this server in System X?"
- COALESCE fallback chains try sources in priority order (AD then ConfigMgr then Snow then SolarWinds)
- Complex columns like Location use deeply nested IF/LOOKUPVALUE (~140 lines)
- CONCATENATEX handles many-to-many relationships (Application, IP Addresses, CMDB Status)
- All lookups are explicit (no relationships) -- LOOKUPVALUE is the workhorse
