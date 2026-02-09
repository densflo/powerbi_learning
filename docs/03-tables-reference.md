# 03 — Tables Reference

## Summary

The TRoI model contains **61 tables** organised across 9 query groups, plus 3 diagnostics expressions and 1 expression-table. This includes 58 import (M query) tables and 3 DAX calculated tables.

Additionally, the `Ops Theater RAETSMARINE Apps` expression materialises as a table (defined in `expressions.tmdl` rather than in the `tables/` folder) and the `ApplicationCataloguePath` parameter is used by the Application Catalogue queries.

---

## Hosting Platforms

### VMware VMs
- **Source**: SharePoint Excel (RVTools `vInfo` sheet) — 3 vCenter exports combined (SVCENT0005, AVS UK South, AVS UK West)
- **Key Columns**: `Name` (server name from mapping), `VM` (VM name, uppercased), `Host`
- **Key Processing**: Filters to `Connection state = "connected"`, joins with `Mapping VM to ServerName` for friendly names, adds `Network` column, deduplicates by VM name (keeps most recent PowerOn)
- **Columns** (85): Name, VM, Powerstate, Template, SRM Placeholder, Config status, DNS Name, Connection state, Guest state, Heartbeat, Consolidation Needed, PowerOn, Suspended To Memory, Suspend time, Suspend Interval, Creation date, Change Version, CPUs, Overall Cpu Readiness, Memory (MB), Active Memory, NICs, Disks, Total disk capacity MiB, Fixed Passthru HotPlug, min Required EVC Mode Key, Latency Sensitivity, Op Notification Timeout, EnableUUID, CBT, Primary IP Address, Network, Network #1–#8, Num Monitors, Video Ram KiB, Resource pool, Folder ID, Folder, vApp, DAS protection, FT State, FT Role, FT Latency, FT Bandwidth, FT Sec. Latency, Vm Failover In Progress, Provisioned MiB, In Use MiB, Unshared MiB, HA Restart Priority, HA Isolation Response, HA VM Monitoring, Cluster rule(s), Cluster rule name(s), Boot Required, Boot delay, Boot retry delay, Boot retry enabled, Boot BIOS setup, Reboot PowerOff, EFI Secure boot, Firmware, HW version, HW upgrade status, HW upgrade policy, HW target, Path, Log directory, Snapshot directory, Suspend directory, Annotation, Backup Status, Last Backup, XdConfig, Datacenter, Cluster, Host, OS according to the configuration file, OS according to the VMware Tools, Customization Info, Guest Detailed Data, VM ID, SMBIOS UUID, VM UUID, VI SDK Server type, VI SDK API Version, VI SDK Server, VI SDK UUID

### VMware Hosts
- **Source**: SharePoint Excel (RVTools `vHost` sheet) — same 3 vCenter exports
- **Key Columns**: `Name`, `Host`, `Location`
- **Purpose**: ESXi host information, used for host-level lookups (Location, Cluster)

### VMware Disks
- **Source**: SharePoint Excel (RVTools `vDisk` sheet)
- **Key Columns**: `VM`, `Host`, `Capacity MiB`
- **Purpose**: Disk capacity per VM, used in `_All Devices[Disk Capacity]` calculation

### VMware CD
- **Source**: SharePoint Excel (RVTools `vCD` sheet)
- **Key Columns**: `VM`, `Host`
- **Purpose**: Mounted CD/DVD media detection for `_All Devices[Mounted Media]`

### VMware Network
- **Source**: SharePoint Excel (RVTools `vNetwork` sheet)
- **Key Columns**: `VM`, `Host`, `Adapter`
- **Purpose**: Network adapter info, used for E1000 adapter detection

### VMware Snapshots
- **Source**: SharePoint Excel (RVTools `vSnapshot` sheet)
- **Key Columns**: `Name`, `Host`
- **Purpose**: Snapshot presence detection for `_All Devices[Snapshots]`

### VMware Tools
- **Source**: SharePoint Excel (RVTools `vTools` sheet)
- **Key Columns**: `VM`, `Host`, `Tools Version`
- **Purpose**: VMware Tools version lookup

### Azure VMs
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `IPAddress`, `Location`, `PowerState`, `Cores`, `Memory`
- **Purpose**: Azure VM inventory for the base UNION in `_All Devices`

### Azure Disks
- **Source**: SharePoint Excel
- **Key Columns**: `managedBy`, `diskSizeGB`
- **Purpose**: Azure disk capacity, used in `_All Devices[Disk Capacity]`

### Azure VM Sizes
- **Source**: SharePoint Excel
- **Purpose**: Azure VM size reference data

### Azure Arc Machines
- **Source**: SharePoint Excel
- **Purpose**: Azure Arc-enrolled machines inventory

### Azure Arc SQL Instances
- **Source**: SharePoint Excel
- **Key Columns**: `Full Instance Name`
- **Purpose**: Central hub for SQL instance relationships (6 tables relate to this via Full Instance Name)

### Azure Arc SQL Databases
- **Source**: SharePoint Excel
- **Key Columns**: `Full Instance Name`
- **Purpose**: SQL databases discovered via Azure Arc

### Azure SQL Virtual Machines
- **Source**: SharePoint Excel
- **Purpose**: Azure SQL VM inventory

### Azure VirtualNetworks
- **Source**: SharePoint Excel
- **Key Columns**: `name`, `subnetAddressPrefix`
- **Purpose**: Azure vNet data, used for `_All Devices[Azure vNet]` lookup

### SolarWinds VMs
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Virtualization Platform`
- **Purpose**: Hyper-V VM detection — filtered to `Virtualization Platform = "Hyper-V"` for `_All Devices` inclusion

---

## Asset Management & Inventory

### Active Directory Computers
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Domain`, `OS`
- **Purpose**: AD computer objects — first source in COALESCE chains for Domain and Operating System

### ConfigMgr All Devices
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Operating System`, `Manufacturer`, `Model`
- **Purpose**: SCCM device inventory — second source in COALESCE for OS, Manufacturer, Model

### ConfigMgr Encryption Status
- **Source**: SharePoint Excel
- **Key Columns**: `Name`
- **Purpose**: BitLocker/encryption status per device — used for `_All Devices[Encrypted Drives]`

### Snow Computers
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Domain`, `Operating System`, `Manufacturer`, `Model`, `Hypervisor`, `Processor cores`, `Memory`
- **Purpose**: Snow Software asset data — used for physical device detection (Hypervisor = "") and as fallback source for many columns

### Snow SQL Servers
- **Source**: SharePoint Excel
- **Purpose**: SQL Server instances from Snow Software

### Application Catalogue Working Catalogue
- **Source**: SharePoint Excel (via `ApplicationCataloguePath` parameter)
- **Key Columns**: `Application Reference`, `Expected DC Handling`, `Proposed DC Handling`, `Business or Technology`, `Legal Entity Usage`, `IBS`
- **Purpose**: Master application catalogue with migration paths, used heavily in `_All Projects Scope` and `_All Applications by Server`

### Application Catalogue Wave Plan
- **Source**: SharePoint Excel (via `ApplicationCataloguePath` parameter)
- **Key Columns**: `Application Reference`, `Wave`
- **Purpose**: Migration wave assignments per application

### Application Catalogue Discrepancies
- **Source**: SharePoint Excel (via `ApplicationCataloguePath` parameter)
- **Purpose**: Data quality — discrepancies between the application catalogue and the data model

### Application Catalogue Missing From Data Model
- **Source**: SharePoint Excel (via `ApplicationCataloguePath` parameter)
- **Purpose**: Data quality — applications in the catalogue not found in the data model

### ServiceNow CMDB
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Install Status`, `Environment`, `Application name`, `Owned by`
- **Purpose**: CMDB records — used for `In CMDB` flag, CMDB Status, Environment, Owner

### ServiceNow Decommission Change Requests
- **Source**: SharePoint Excel
- **Key Columns**: `Number`, `Description`
- **Purpose**: Decom change requests — searched via `CONTAINSSTRING` for server names in `_All Devices[Decom CHG Number]`

### ServiceNow Decommission Service Requests
- **Source**: SharePoint Excel
- **Purpose**: Decom service requests

### ServiceNow MSSQL Databases
- **Source**: SharePoint Excel
- **Key Columns**: `Full Instance Name`
- **Purpose**: SQL databases from ServiceNow, relates to Azure Arc SQL Instances

### ServiceNow MSSQL Instances
- **Source**: SharePoint Excel
- **Key Columns**: `Full Instance Name`
- **Purpose**: SQL instances from ServiceNow, relates to Azure Arc SQL Instances

### Ops Theater RAETSMARINE Apps
- **Source**: **Web scraping** — `http://spdoca0001.raetsmarine.local/servers_func.html`
- **Columns**: Host, IP, Env, Branch, Application, Component, Contact, Comment
- **Purpose**: Internal Ops Theater application-to-server mapping from an intranet HTML table

---

## Monitoring Tools

### Qualys Assets
- **Source**: SharePoint Excel
- **Key Columns**: `Name`
- **Purpose**: Qualys vulnerability scanner inventory — used for `In Qualys` flag

### Symantec Endpoint Protection Devices
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Security Status`
- **Purpose**: SEP antivirus status — used for `In SEP` flag and `SEP Status` lookup

### QRadar Log Sources
- **Source**: SharePoint Excel
- **Purpose**: SIEM log source inventory

### Panaseer All Devices
- **Source**: SharePoint Excel
- **Key Columns**: `Name`
- **Purpose**: Panaseer security posture data — used for `In Panaseer` flag

### SolarWinds Nodes
- **Source**: SharePoint Excel
- **Key Columns**: `Name`, `Node Category`, `DeviceType`, `Domain`, `Machine Type`, `Manufacturer`, `Model`
- **Purpose**: SolarWinds network monitoring inventory — filtered for server types in `_All Devices` inclusion; also used for Domain, OS, Manufacturer, Model fallback

---

## Mapping Tables

### Mapping Application to Application Catalogue
- **Key Columns**: `Name`, `Name (Application Catalogue)`, `Reference`, `KS Comments`
- **Purpose**: Maps TRoI application names to Application Catalogue names and references

### Mapping Diamond IP IPControl Subnet to Location
- **Key Columns**: `Container`, `Location`
- **Purpose**: Maps Diamond IP subnet containers to physical locations

### Mapping DomainFQDN to DomainNetBIOS
- **Key Columns**: `NetBIOS`, FQDN columns
- **Purpose**: FQDN-to-NetBIOS domain name translation

### Mapping DomainNetBIOS to DomainFQDN
- **Key Columns**: `NetBIOS`, FQDN columns
- **Purpose**: NetBIOS-to-FQDN domain name translation

### Mapping Location
- **Purpose**: Location normalisation/reference data

### Mapping Manual Missing Subnets
- **Key Columns**: `Location`
- **Purpose**: Manual overrides for subnets not found in automated sources

### Mapping Manual Server To Location
- **Key Columns**: `Name`, `Location`
- **Purpose**: Manual server-to-location mapping — last fallback in `_All Devices[Location]` chain

### Mapping Server to Application
- **Key Columns**: `Server`, `Application`
- **Purpose**: Core server-to-application mapping used by `_All Devices[Application]`, `_All Projects Scope`, and `_All Applications by Server`

### Mapping VLAN to Application
- **Purpose**: VLAN-to-application mapping

### Mapping VM to ServerName
- **Source**: SharePoint Excel — `https://amlinonline.sharepoint.com/.../Mapping-VM-ServerName.xlsx`
- **Key Columns**: `VM`, `Name`
- **Purpose**: Maps VMware VM names to friendly server names (both uppercased)

---

## Merged Tables

### Merged IP Addresses
- **Key Columns**: `Name`, `IP Address`, `IP Address (Decimal)`, `Subnet`
- **Purpose**: Combined IP address data from multiple sources, used for `_All Devices[IP Address]` and `_All Devices[IP Addresses]`

### Merged Subnets
- **Key Columns**: `Subnet`, `Location`
- **Purpose**: Combined subnet data with location mapping — used in `_All Devices[Location]` fallback chain

### Merged Subnets Expanded
- **Key Columns**: `Subnet`, `IP Address`, `IP Address (Decimal)`
- **Purpose**: Every individual IP address in each subnet (generated via `ExpandSubnet` function) — used for `_All Devices[IP Subnet]` lookup

### Merged SQL Databases
- **Key Columns**: `Server Name`, `Full Instance Name`, `Database Name`, `Application`, `SQL Version`, `SQL Edition`, `Source`, `Server In Model`
- **Purpose**: Combined SQL database records from Azure Arc, ServiceNow, and DB Inventory System — displayed on the Databases report page

### Merged SQL Instances
- **Key Columns**: `Full Instance Name`
- **Purpose**: Combined SQL instance records from multiple sources

---

## Network

### Diamond IP IPControl Subnets
- **Key Columns**: `Container`
- **Purpose**: Subnet data from Diamond IP IPControl IPAM system

### Network Team Global IP Address Allocation
- **Key Columns**: `Subnet`
- **Purpose**: IP address allocation data from the network team

### Network Team Network Switch Traffic
- **Key Columns**: `IP Address`, `VLAN`
- **Purpose**: Switch traffic data — used for `_All Devices[Network]` VLAN lookup

---

## Other Teams

### Citrix Team Published Applications
- **Purpose**: Published Citrix applications inventory — displayed on the Citrix Published Applications report page

### DR Manager Important Business Services
- **Purpose**: Disaster recovery important business services

### Environments Team Environment Management Database
- **Key Columns**: `Server Name`, `Application Name`
- **Purpose**: Environment management data — used for `_All Devices[Application (Environment Management Database)]`

---

## DB Inventory

### DB Inventory System Databases
- **Key Columns**: `Full Instance Name`
- **Purpose**: SQL databases from the DB Inventory System, relates to Azure Arc SQL Instances

### DB Inventory System Instances
- **Key Columns**: `Full Instance Name`
- **Purpose**: SQL instances from the DB Inventory System, relates to Azure Arc SQL Instances

---

## Calculated Tables (DAX)

### _All Devices
- **Type**: DAX calculated table
- **Base Columns**: `Name`, `VM` (from partition source)
- **Calculated Columns**: 30+ columns (see [05 — DAX Formulas](05-dax-formulas.md))
- **Measures**: 10 measures
- **Purpose**: Core unified device list — the heart of the entire model

### _All Projects Scope
- **Type**: DAX calculated table
- **Base Columns**: Name, VM Name, Device Type, Power State, Location, CMDB Status, Decom CHG, Application, IP Address, Environment (from partition source)
- **Calculated Columns**: Application (Application Catalogue), Application Reference, Application Name Match, Wave, Expected DC Migration Path, Proposed DC Migration Path, KS Comments, Network, Domain, Legal Entity Usage, Type
- **Measures**: Applications, Servers, Apps (Apps Cat)
- **Purpose**: Server-application combinations for project/migration planning — excludes "Client Endpoint" and AVS/New ALZ locations

### _All Applications by Server
- **Type**: DAX calculated table
- **Base Columns**: Name, VM Name, Device Type, Power State, Operating System, End of Life, Location, CMDB Status, Decom CHG, Application, IP Address, Environment (from partition source)
- **Calculated Columns**: Application (Application Catalogue), Application Reference (Application Catalogue), IBS, Owner, Proposed DC Migration Path, Type
- **Purpose**: Server-application combinations with End of Life flag (Windows 2008/2012), IBS, Owner, and migration data
