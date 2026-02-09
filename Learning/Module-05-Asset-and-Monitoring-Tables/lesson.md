# Module 05: Asset and Monitoring Tables

## What You'll Learn
The 18 tables from AD, ConfigMgr, Snow, ServiceNow, Qualys, SEP, and more.

---

## 1. Why Multiple Systems Track the Same Servers

Different IT teams use different tools. Each tool sees the server from its own perspective:

- **Active Directory** -- Identity and authentication (Domain, OS, Last Logon)
- **ConfigMgr (SCCM)** -- Software deployment and configuration (Manufacturer, Model, Encryption)
- **Snow Software** -- Software asset management (License compliance, Hardware details)
- **ServiceNow CMDB** -- IT service management (Environment, Install Status, Owner, Application name)
- **Qualys** -- Vulnerability scanning (security posture)
- **SEP (Symantec)** -- Antivirus/endpoint protection (Security Status)
- **QRadar** -- SIEM (security event logging)
- **Panaseer** -- Security posture aggregation
- **SolarWinds Nodes** -- Network monitoring (DeviceType, Node Category)

No single system has a complete picture -- that is why TRoI exists: to merge them all into one unified view.

---

## 2. Asset Management Tables (5)

### Active Directory Computers

Columns: Name, Enabled, Domain, IP Address, Last Logon, Logon Age, OS.

Provides domain membership and last logon information. In `_All Devices`, this is the first source tried for Domain and OS (via COALESCE fallback).

### ConfigMgr All Devices

Columns: Name, Operating System, Manufacturer, Model.

Provides hardware details and OS from SCCM. Second source for OS in the COALESCE chain.

### ConfigMgr Encryption Status

Columns: Name.

Used purely as a boolean check -- if a server Name appears in this table, it has BitLocker encryption enabled (`Encrypted Drives = TRUE` in `_All Devices`).

### Snow Computers

Columns: Name, Domain, Operating System, Manufacturer, Model, Processor cores, Memory, Hypervisor.

Provides hardware specs and software licensing data. The `Hypervisor` column is important: it is used to filter physical servers (`Hypervisor = ""`) for the Snow EXCEPT block in `_All Devices`. Servers with a non-blank Hypervisor value are virtual machines already captured by another source.

### ServiceNow CMDB

Columns: Name, Environment, Install Status, Application name, Owned by.

The richest asset data -- provides environment classification, operational status, and application ownership. Multiple CMDB records per server are possible (one per application), which is why `_All Devices` uses `CONCATENATEX` for the CMDB Status column rather than a simple `LOOKUPVALUE`.

---

## 3. ServiceNow Decommission Tables (2)

### ServiceNow Decommission Change Requests

Contains CHG numbers and descriptions for decommission changes. `_All Devices` searches the Description field using `CONTAINSSTRING` to match server names to decommission CHG numbers. This is a text-search approach because the Description field contains free-text references to server names rather than a structured foreign key.

### ServiceNow Decommission Service Requests

Service request records for decommissions. Complements the change request data.

---

## 4. Application Catalogue Tables (4)

These 4 tables all source from the same SharePoint Excel file, controlled by the `ApplicationCataloguePath` parameter. When the file moves to a new SharePoint location, update the parameter and all 4 tables will pick up the new path.

### Application Catalogue Wave Plan

Migration wave assignments per application reference. Used for project planning -- which applications migrate in which wave.

### Application Catalogue Working Catalogue

Master application catalogue. Key columns: Expected/Proposed DC Handling, Legal Entity Usage, Business or Technology classification. This is the authoritative list of applications.

### Application Catalogue Discrepancies

Gaps between the data model and the catalogue. Highlights where the inventory disagrees with the official application catalogue.

### Application Catalogue Missing From Data Model

Applications in the catalogue that do not appear in the data model. Flags coverage gaps -- applications that should have associated servers but none are found.

---

## 5. Monitoring Tools (5)

These tables drive the "gap analysis" -- answering the question: "Is every server monitored by every required tool?"

### Qualys Assets

Qualys vulnerability scanner inventory. Key column: Name. Used for the "In Qualys" boolean flag in `_All Devices`.

### Symantec Endpoint Protection Devices

SEP antivirus agents. Key columns: Name, Security Status. Used for the "In SEP" boolean flag and the "SEP Status" lookup in `_All Devices`.

### QRadar Log Sources

SIEM log sources. Key column: Name. Used for security event coverage analysis.

### Panaseer All Devices

Security posture aggregation. Key column: Name. Used for the "In Panaseer" boolean flag in `_All Devices`.

### SolarWinds Nodes

Network-monitored nodes. Key columns: Name, Node Category, DeviceType, Domain, Manufacturer, Model, Machine Type.

This table serves a dual purpose:
1. Used for the "In SolarWinds Nodes" boolean flag.
2. Acts as a fallback source for Domain, OS, Manufacturer, and Model in the COALESCE chains. If AD and ConfigMgr do not have the data, SolarWinds Nodes is tried next.

---

## 6. The Ops Theater Expression-Table

`Ops Theater RAETSMARINE Apps` is not stored in the `tables/` folder. It is defined in `expressions.tmdl` as an expression-table that web-scrapes `http://spdoca0001.raetsmarine.local/servers_func.html` for server/application data.

Key considerations:
- Requires intranet access to the `raetsmarine.local` domain.
- Fragile to HTML structure changes on the source page.
- If the scrape fails, the table will be empty but will not block the rest of the model from refreshing.

---

## 7. The Boolean Flag Pattern

Each monitoring tool creates a boolean flag column in `_All Devices`. The pattern is consistent across all of them:

```dax
column 'In Qualys' =
    IF(
        NOT(ISBLANK(LOOKUPVALUE('Qualys Assets'[Name], 'Qualys Assets'[Name], [Name]))),
        TRUE,
        FALSE
    )
```

Translation: "If I can find this server's Name in the Qualys table, return TRUE (monitored). Otherwise return FALSE (gap)."

This same pattern repeats for:
- `In SEP` -- looks up in `Symantec Endpoint Protection Devices`
- `In SolarWinds Nodes` -- looks up in `SolarWinds Nodes`
- `In Panaseer` -- looks up in `Panaseer All Devices`
- `In ConfigMgr` -- looks up in `ConfigMgr All Devices`
- `In AD` -- looks up in `Active Directory Computers`

These boolean flags are the foundation for gap analysis measures (covered in Module 11). A measure like `Not In Qualys` simply counts how many servers have `In Qualys = FALSE`.

---

## 8. Hands-On Exercise

1. In **Data view**, click on **`Active Directory Computers`** -- look at the columns available (Name, Domain, OS, Last Logon).
2. Now click on **`ConfigMgr All Devices`** -- notice different columns for the same servers (Manufacturer, Model).
3. Click on **`Snow Computers`** -- see the hardware data (Manufacturer, Model, Processor cores, Memory).
4. Click on **`ServiceNow CMDB`** -- see operational data (Environment, Install Status, Application name).
5. Pick a specific server name you can see in AD, then search for it in ConfigMgr, Snow, and CMDB. Notice what each system knows about the same server and what each system is missing.
6. Click on **`Qualys Assets`** -- this table is used purely for boolean coverage checking. The column list is minimal because only Name matters for the lookup.
7. In **Model view**, notice that the monitoring tables (Qualys, SEP, QRadar, Panaseer) have no relationships to other tables. Their data flows into `_All Devices` via LOOKUPVALUE in calculated columns, not through the relationship model.

---

## 9. Key Takeaways

- Different IT systems provide different perspectives on the same servers. No single source has the full picture.
- Asset tables (AD, ConfigMgr, Snow, CMDB) contribute specific columns via `LOOKUPVALUE` and `COALESCE` fallback chains in `_All Devices`.
- Monitoring tables (Qualys, SEP, QRadar, Panaseer, SolarWinds Nodes) create boolean "In X" flags for gap analysis.
- The boolean flag pattern (`IF(NOT(ISBLANK(LOOKUPVALUE(...))), TRUE, FALSE)`) is the single most repeated DAX pattern in the model.
- Application Catalogue tables are parameterized -- all 4 are controlled by the `ApplicationCataloguePath` parameter.
- SolarWinds Nodes serves double duty: both a monitoring flag source and a fallback for asset data in COALESCE chains.
- TRoI merges all of these sources so that every server has a single, consolidated row with the best available data from across all systems.
