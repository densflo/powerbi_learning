# 05 — DAX Formulas

This is the most critical documentation file. It covers the DAX partition sources, calculated columns, and measures that power the TRoI model.

---

## `_All Devices` — Partition Source (Core UNION Query)

The `_All Devices` table is a **calculated table** built by combining servers from multiple source systems while deduplicating.

### Step-by-Step Logic

```dax
-- Step 1: Base set — VMware VMs, VMware Hosts, and Azure VMs
VAR AllDevices =
    UNION(
        SELECTCOLUMNS('VMware VMs', "Name", ...[Name], "VM", ...[VM]),
        SELECTCOLUMNS('VMware Hosts', "Name", ...[Name], "VM", BLANK()),
        SELECTCOLUMNS('Azure VMs', "Name", ...[Name], "VM", BLANK())
    )

-- Step 2: Add Hyper-V VMs from SolarWinds (that aren't already in AllDevices)
VAR SolarWindsVMsFiltered =
    EXCEPT(
        SELECTCOLUMNS(
            FILTER('SolarWinds VMs', [Virtualization Platform] = "Hyper-V"),
            "Name", ...[Name], "VM", BLANK()
        ),
        SELECTCOLUMNS(AllDevices, "Name", [Name], "VM", BLANK())
    )

-- Step 3: Add non-hypervisor Snow Computers (bare metal/physical)
VAR SnowComputersFiltered =
    EXCEPT(
        SELECTCOLUMNS(
            FILTER('Snow Computers', [Hypervisor] = ""),
            "Name", ...[Name], "VM", BLANK()
        ),
        SELECTCOLUMNS(AllDevices, "Name", [Name], "VM", BLANK())
    )

-- Step 4: Add server-type SolarWinds Nodes (excluding names ending in "-R")
VAR SolarWindsNodesFiltered =
    EXCEPT(
        SELECTCOLUMNS(
            FILTER('SolarWinds Nodes',
                ((Node Category = "Network" && DeviceType = "Server") ||
                 (Node Category = "Other" && DeviceType = "Server") ||
                 (Node Category = "Server")) &&
                NOT RIGHT([Name], 2) = "-R"
            ),
            "Name", ...[Name], "VM", BLANK()
        ),
        SELECTCOLUMNS(AllDevices, "Name", [Name], "VM", BLANK())
    )

-- Step 5: Combine all sets
VAR CombinedDevices =
    UNION(AllDevices, SolarWindsVMsFiltered, SnowComputersFiltered, SolarWindsNodesFiltered)

-- Step 6: Deduplicate — if a Name has a row with a VM and a row without, keep only the VM row
VAR FilteredDevices =
    FILTER(CombinedDevices,
        NOT ([VM] = BLANK() &&
             COUNTROWS(FILTER(CombinedDevices, [Name] = EARLIER([Name]) && NOT(ISBLANK([VM])))) > 0
        )
    )

RETURN SUMMARIZE(FilteredDevices, [Name], [VM])
```

### Key Points
- Output columns: `Name` (server name) and `VM` (VMware VM name or BLANK for non-VMware)
- VMware VMs carry both `Name` and `VM`; all other sources have `VM = BLANK()`
- The dedup filter in Step 6 ensures that if a server appears from both a VMware source (with VM name) and another source (without), only the VMware row is kept
- The `SolarWindsNodesFiltered` additionally excludes names ending in `-R` (likely redundant/backup nodes)

### Discrepancy: DAX Query File vs Partition Source

The DAX query file (`_All Devices.dax`) **comments out** `SnowComputersFiltered`:
```dax
VAR CombinedDevices = UNION(AllDevices, SolarWindsVMsFiltered, //SnowComputersFiltered, SolarWindsNodesFiltered)
```

But the **partition source** in the `.tmdl` **includes** it:
```dax
VAR CombinedDevices = UNION(AllDevices, SolarWindsVMsFiltered, SnowComputersFiltered, SolarWindsNodesFiltered)
```

> **[REVIEW]**: The `.dax` file appears to be a development/testing version where Snow Computers was temporarily excluded. The live partition source includes Snow Computers. The `.dax` file should be treated as a reference only.

---

## `_All Devices` — Calculated Columns (30+)

### Device Type
```dax
VAR VMName = '_All Devices'[Name]
RETURN SWITCH(TRUE(),
    NOT(ISBLANK(LOOKUPVALUE('VMware VMs'[Name], ...[Name], VMName))), "VMware",
    NOT(ISBLANK(LOOKUPVALUE('Azure VMs'[Name], ...[Name], VMName))), "Azure",
    NOT(ISBLANK(CALCULATE(SELECTEDVALUE('SolarWinds VMs'[Name]),
        'SolarWinds VMs'[Virtualization Platform] = "Hyper-V", ...[Name] = VMName))), "Hyper-V",
    [Manufacturer] IN {"Dell Inc.", "HP", "HPE", "IBM"}, "Physical",
    BLANK()
)
```
**Purpose**: Classifies each device as VMware, Azure, Hyper-V, or Physical based on which source system contains it. Physical detection uses manufacturer names.

---

### Location
**Purpose**: Determines the physical/logical location of each device using a 5-tier fallback chain.

**Fallback Priority**:
1. VMware VMs `[Location]` (via VM name)
2. Azure VMs `[Location]` (via Name)
3. VMware Hosts `[Location]` (only if Application = "VMware ESXi")
4. Merged Subnets `[Location]` (via IP Subnet)
5. Mapping Manual Server To Location `[Location]` (manual override/last resort)

> **[REVIEW]**: This column contains deeply nested repeated IF/LOOKUPVALUE logic (~140 lines). The same 5-tier fallback is duplicated across three branches (VMware path, Azure path, default path). This could be refactored into a cleaner pattern but functions correctly as-is.

---

### IP Address
```dax
VAR AzureIP = LOOKUPVALUE('Azure VMs'[IPAddress], ...[Name], [Name])
VAR VMwareIPString = CONCATENATEX(FILTER('VMware VMs', ...[VM] = [VM]), ...[Primary IP Address], ", ")
VAR MergedIPString = CONCATENATEX(
    FILTER('Merged IP Addresses', ...[Name] = [Name]
        && LEFT([IP Address], 3) <> "192"
        && LEFT([IP Address], 3) <> "169"),
    ...[IP Address], ", ")
RETURN SWITCH(TRUE(),
    NOT ISBLANK(AzureIP), AzureIP,
    NOT ISBLANK(VMwareIPString), VMwareIPString,
    NOT ISBLANK(MergedIPString), MergedIPString,
    BLANK()
)
```
**Purpose**: Primary IP address with priority: Azure IP → VMware IPs → Merged IPs. Excludes 192.x.x.x (private) and 169.x.x.x (APIPA) from Merged IPs.

---

### IP Subnet
```dax
LOOKUPVALUE('Merged Subnets Expanded'[Subnet], ...[IP Address], [IP Address])
```
**Purpose**: Finds the subnet containing the device's IP address by looking up in the expanded subnet table.

---

### IP Addresses (all)
```dax
-- Concatenates ALL known IP addresses for a device from Merged IP Addresses
CONCATENATEX(DISTINCT(filtered IPs), [IP Address], ", ", [IP Address], ASC)
```
**Purpose**: Complete list of all known IP addresses (unlike `IP Address` which returns only the primary).

---

### Network
```dax
VAR VMwareNetwork = CALCULATE(FIRSTNONBLANK('VMware VMs'[Network], ...),
    FILTER('VMware VMs', ...[Name] = EARLIER([Name])))
RETURN COALESCE(
    LOOKUPVALUE('Network Team Network Switch Traffic'[VLAN], ...[IP Address], [IP Address]),
    VMwareNetwork,
    BLANK()
)
```
**Purpose**: Network/VLAN. Priority: Switch Traffic VLAN (from IP) → VMware Network name.

---

### Domain
```dax
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[Domain], ...[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Domain], ...[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Domain], ...[Name], [Name])
)
```
**Purpose**: Domain name from AD → Snow → SolarWinds.

---

### Operating System
```dax
COALESCE(
    LOOKUPVALUE('Active Directory Computers'[OS], ...[Name], [Name]),
    LOOKUPVALUE('ConfigMgr All Devices'[Operating System], ...[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Operating System], ...[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Machine Type], ...[Name], [Name])
)
```
**Purpose**: OS from AD → ConfigMgr → Snow → SolarWinds.

---

### Manufacturer
```dax
COALESCE(
    LOOKUPVALUE('Snow Computers'[Manufacturer], ...[Name], [Name]),
    LOOKUPVALUE('ConfigMgr All Devices'[Manufacturer], ...[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Manufacturer], ...[Name], [Name]),
    BLANK()
)
```
**Purpose**: Hardware manufacturer from Snow → ConfigMgr → SolarWinds.

---

### Model
```dax
COALESCE(
    LOOKUPVALUE('Snow Computers'[Model], ...[Name], [Name]),
    LOOKUPVALUE('ConfigMgr All Devices'[Model], ...[Name], [Name]),
    LOOKUPVALUE('SolarWinds Nodes'[Model], ...[Name], [Name]),
    BLANK()
)
```
**Purpose**: Hardware model from Snow → ConfigMgr → SolarWinds.

---

### Power State
```dax
SWITCH(TRUE(),
    [Device Type] = "Physical", BLANK(),
    [Device Type] = "VMware",
        SWITCH(LOOKUPVALUE(...[Powerstate], ...), "poweredOn", "ON", "poweredOff", "OFF", <raw value>),
    [Device Type] = "Azure",
        SWITCH(LOOKUPVALUE(...[PowerState], ...), "VM running", "ON", "VM deallocated", "OFF", <raw value>),
    BLANK()
)
```
**Purpose**: Normalises power states to "ON"/"OFF" across platforms. Physical devices return BLANK.

---

### Application
```dax
CONCATENATEX(
    DISTINCT(CALCULATETABLE(VALUES('Mapping Server to Application'[Application]),
        ...[Server] = [Name])),
    ...[Application], ", ", ...[Application], ASC
)
```
**Purpose**: Comma-separated list of all applications mapped to this server (sorted alphabetically).

---

### Environment
```dax
CALCULATE(FIRSTNONBLANK('ServiceNow CMDB'[Environment], ...),
    FILTER('ServiceNow CMDB', ...[Name] = EARLIER([Name])))
```
**Purpose**: Environment from ServiceNow CMDB.

---

### Cores
```dax
COALESCE(
    LOOKUPVALUE('VMware VMs'[CPUs], ...[VM], [VM]),
    LOOKUPVALUE('Azure VMs'[Cores], ...[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Processor cores], ...[Name], [Name]),
    BLANK()
)
```
**Purpose**: CPU core count from VMware → Azure → Snow.

---

### Memory
```dax
COALESCE(
    LOOKUPVALUE('VMware VMs'[Memory], ...[VM], [VM]),
    LOOKUPVALUE('Azure VMs'[Memory], ...[Name], [Name]),
    LOOKUPVALUE('Snow Computers'[Memory], ...[Name], [Name]),
    BLANK()
)
```
**Purpose**: Memory from VMware → Azure → Snow.

---

### Disk Capacity
```dax
VAR VMwareTotalMiB = CALCULATE(SUM('VMware Disks'[Capacity MiB]), ...[VM] = VMName)
VAR AzureTotalGiB = CALCULATE(SUM('Azure Disks'[diskSizeGB]), ...[managedBy] = ServerName)
VAR TotalMiB = VMwareTotalMiB + (AzureTotalGiB * 1024)
-- Formats as TB/GB/MB depending on size
```
**Purpose**: Combined disk capacity from VMware (MiB) and Azure (GiB), formatted as human-readable TB/GB/MB. Returns BLANK for non-VMware/non-Hyper-V devices without a VM name.

---

### Boolean "In [System]" Flags

All follow the same pattern:
```dax
IF(NOT(ISBLANK(LOOKUPVALUE('<Table>'[Name], '<Table>'[Name], [Name]))), TRUE, FALSE)
```

| Column | Source Table | Purpose |
|--------|-------------|---------|
| `In AD` | Active Directory Computers | Present in Active Directory |
| `In CM` | ConfigMgr All Devices | Present in ConfigMgr/SCCM |
| `In CMDB` | ServiceNow CMDB | Present in ServiceNow CMDB |
| `In Snow` | Snow Computers | Present in Snow Software |
| `In Qualys` | Qualys Assets | Present in Qualys vulnerability scanner |
| `In SEP` | Symantec Endpoint Protection Devices | Present in Symantec Endpoint Protection |
| `In SolarWinds Nodes` | SolarWinds Nodes | Present in SolarWinds Nodes |
| `In SolarWinds VMs` | SolarWinds VMs | Present in SolarWinds VMs |
| `In Panaseer` | Panaseer All Devices | Present in Panaseer |

---

### Encrypted Drives
```dax
IF(CONTAINS('ConfigMgr Encryption Status', ...[Name], [Name]), TRUE, FALSE)
```
**Purpose**: Whether ConfigMgr reports encrypted drives for this device.

---

### Snapshots
```dax
IF(CONTAINS('VMware Snapshots', ...[Name], [Name]), TRUE, FALSE)
```
**Purpose**: Whether any VMware snapshots exist for this device.

---

### Mounted Media
```dax
IF(CONTAINS('VMware CD', ...[VM], [VM]), TRUE, FALSE)
```
**Purpose**: Whether a CD/DVD is mounted on this VM.

---

### E1000 Adapter
```dax
IF(COUNTROWS(FILTER('VMware Network', ...[VM] = EARLIER([VM]) && ...[Adapter] <> "Vmxnet3")) > 0, TRUE, FALSE)
```
**Purpose**: Whether the VM has any non-Vmxnet3 network adapters (legacy E1000 adapters are a migration concern).

---

### VMware Tools Version
```dax
LOOKUPVALUE('VMware Tools'[Tools Version], ...[VM], [VM])
```

### SEP Status
```dax
LOOKUPVALUE('Symantec Endpoint Protection Devices'[Security Status], ...[Name], [Name])
```

### Host
```dax
LOOKUPVALUE('VMware VMs'[Host], ...[VM], [VM])
```

### Cluster
```dax
LOOKUPVALUE('VMware VMs'[Cluster], ...[VM], [VM])
```

### Owner
```dax
LOOKUPVALUE('ServiceNow CMDB'[Owned by], ...[Name], [Name])
```

### Application (CMDB)
```dax
LOOKUPVALUE('ServiceNow CMDB'[Application name], ...[Name], [Name])
```

### Application (Environment Management Database)
```dax
CONCATENATEX(DISTINCT(...), 'Environments Team Environment Management Database'[Application Name], ", ", ..., ASC)
```
**Purpose**: Applications from the Environments Team database (comma-separated).

### Operating System (VMware Tools)
```dax
LOOKUPVALUE('VMware VMs'[OS according to the VMware Tools], ...[VM], [VM])
```

---

### CMDB Status
```dax
CONCATENATEX(DISTINCT(CALCULATETABLE(VALUES('ServiceNow CMDB'[Install Status]), ...[Name] = [Name])),
    ...[Install Status], ", ", ..., ASC)
```
**Purpose**: Comma-separated list of all CMDB install statuses for this device (a device may have multiple CMDB records).

---

### Decom CHG Number
```dax
VAR MatchedRows = FILTER('ServiceNow Decommission Change Requests',
    CONTAINSSTRING(...[Description], '_All Devices'[Name]))
RETURN IF(COUNTROWS(MatchedRows) > 0,
    CONCATENATEX(DISTINCT(MatchedRows), ...[Number], ", ", ..., ASC), BLANK())
```
**Purpose**: Finds decommission change request numbers where the device name appears in the description text.

---

### Has Matching AVS Server
```dax
VAR IsRomfordOrHemel = [Location] IN {"Romford", "Hemel Hempstead"}
VAR MatchingAVS = CALCULATE(COUNTROWS('_All Devices'),
    FILTER('_All Devices', [Name] = CurrentServer && [Location] IN {"AVS - UK South", "AVS - UK West"}))
RETURN IF(IsRomfordOrHemel && MatchingAVS > 0, TRUE, FALSE)
```
**Purpose**: For on-premises servers (Romford/Hemel Hempstead), checks if the same server name exists in an AVS location — indicating it has been migrated.

---

### Azure vNet
```dax
CONCATENATEX(DISTINCT(CALCULATETABLE(VALUES('Azure VirtualNetworks'[name]),
    ...[subnetAddressPrefix] = [IP Subnet])), ...[name], ", ", ..., ASC)
```
**Purpose**: Azure Virtual Network name(s) associated with the device's IP subnet.

---

## `_All Devices` — Measures (10)

| Measure | DAX | Purpose |
|---------|-----|---------|
| `No. of Devices` | `COUNTA([Name])` | Count of devices in current filter context |
| `Has Mounted Media` | `CALCULATE(COUNTA([Mounted Media]), [Mounted Media] IN {TRUE})` | Count of devices with mounted CD/DVD |
| `Has Encrypted Drives` | `CALCULATE(COUNTA([Encrypted Drives]), [Encrypted Drives] IN {TRUE})` | Count of devices with encrypted drives |
| `Has Snapshots` | `CALCULATE(COUNTA([Snapshots]), [Snapshots] IN {TRUE})` | Count of devices with VMware snapshots |
| `No. of Networks` | `DISTINCTCOUNTNOBLANK([Network])` | Count of distinct networks |
| `Not In Qualys` | `CALCULATE(COUNTA([In Qualys]), [In Qualys] IN {FALSE})` | Devices missing from Qualys |
| `Not In SEP` | `CALCULATE(COUNTA([In SEP]), [In SEP] IN {FALSE})` | Devices missing from SEP |
| `Not In CMDB` | `CALCULATE(COUNTA([In CMDB]), [In CMDB] IN {FALSE})` | Devices missing from CMDB |
| `Not In SWO Nodes` | `CALCULATE(COUNTA([In SolarWinds Nodes]), ... IN {FALSE})` | Devices missing from SolarWinds Nodes |
| `Not In SWO VMs` | `CALCULATE(COUNTA([In SolarWinds VMs]), ... IN {FALSE})` | Devices missing from SolarWinds VMs |

All measures return `0` instead of BLANK when the count is null.

---

## `_All Projects Scope` — Partition Source

```dax
DISTINCT(
    SELECTCOLUMNS(
        GENERATE(
            '_All Devices',
            FILTER('Mapping Server To Application',
                [Server] = '_All Devices'[Name] &&
                [Application] <> "Client Endpoint" &&
                NOT CONTAINSSTRING('_All Devices'[Location], "AVS") &&
                NOT CONTAINSSTRING('_All Devices'[Location], "New ALZ")
            )
        ),
        "Name", ...[Name], "VM Name", ...[VM], "Device Type", ...[Device Type],
        "Power State", ...[Power State], "Location", ...[Location],
        "CMDB Status", ...[CMDB Status], "Decom CHG", ...[Decom CHG Number],
        "Application", ...[Application], "IP Address", ...[IP Address],
        "Environment", ...[Environment]
    )
)
```

**Purpose**: Creates one row per server-application combination for project planning. Filters:
- Excludes "Client Endpoint" applications
- Excludes devices in AVS locations
- Excludes devices in "New ALZ" locations

### Calculated Columns
- **Application (Application Catalogue)**: Maps to catalogue name via `Mapping Application to Application Catalogue`
- **Application Reference**: Maps to catalogue reference
- **Application Name Match**: Compares TRoI app name to catalogue name (exact match)
- **Wave**: From `Application Catalogue Wave Plan`
- **Expected DC Migration Path**: From `Application Catalogue Working Catalogue[Expected DC Handling]`
- **Proposed DC Migration Path**: From `Application Catalogue Working Catalogue[Proposed DC Handling]`
- **Network**: Same COALESCE logic as `_All Devices[Network]`
- **Domain**: Same COALESCE logic as `_All Devices[Domain]`
- **Legal Entity Usage**: From `Application Catalogue Working Catalogue`
- **Type**: Maps "Business Application" → "Business", "Technology Component" → "Technology"
- **KS Comments**: From `Mapping Application to Application Catalogue`

### Measures
- **Applications**: `DISTINCTCOUNTNOBLANK([Application])`
- **Servers**: `DISTINCTCOUNTNOBLANK([Name])`
- **Apps (Apps Cat)**: `DISTINCTCOUNTNOBLANK([Application (Application Catalogue)])`

---

## `_All Applications by Server` — Partition Source

```dax
DISTINCT(
    SELECTCOLUMNS(
        GENERATE(
            '_All Devices',
            FILTER('Mapping Server To Application',
                [Server] = '_All Devices'[Name] &&
                NOT CONTAINSSTRING([Application], "Client Endpoint")
            )
        ),
        "Name", ..., "VM Name", ..., "Device Type", ..., "Power State", ...,
        "Operating System", ...,
        "End of Life", OR(CONTAINSSTRING([Operating System], "2008"),
                          CONTAINSSTRING([Operating System], "2012")),
        "Location", ..., "CMDB Status", ..., "Decom CHG", ...,
        "Application", ..., "IP Address", ..., "Environment", ...
    )
)
```

**Purpose**: Similar to `_All Projects Scope` but:
- Does NOT exclude AVS/New ALZ locations
- Includes `Operating System` and `End of Life` flag
- `End of Life` = TRUE if OS contains "2008" or "2012"

### Calculated Columns
- **Application (Application Catalogue)**: Same mapping as Projects Scope
- **Application Reference**: Same mapping
- **IBS**: From `Application Catalogue Working Catalogue`
- **Owner**: From `ServiceNow CMDB[Owned by]`
- **Proposed DC Migration Path**: Same mapping
- **Type**: Maps "Business Application" → "Business Application", "Technology Component" → "Infrastructure" (note: slightly different mapping than Projects Scope)
