# 02 — Data Sources

## Overview

The TRoI model imports data from multiple enterprise systems via Power Query (M). Sources include SharePoint-hosted Excel files, Azure APIs, internal web pages, and SharePoint lists.

### Parameter

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `ApplicationCataloguePath` | `https://amlinonline.sharepoint.com/sites/DataCentreStrategy/Shared%20Documents/General/DC%20Migration/03.%20Assessment/Application%20mapping/Apps%20Cat%20updates%20for%20DC%20Work%20-%20Working%20Version%20-%20Not%20baselined.xlsx` | SharePoint URL for the Application Catalogue Excel workbook |

---

## Hosting Platforms

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| VMware VMs | SharePoint Excel (RVTools export) | `https://amlinonline.sharepoint.com/.../RVTools_export_all_SVCENT0005.xlsx`, `...AVSUKSouth.xlsx`, `...AVSUKWest.xlsx` — combined from 3 vCenter exports |
| VMware Hosts | SharePoint Excel (RVTools vHost sheet) | Same SharePoint site, RVTools vHost sheet from same 3 workbooks |
| VMware Disks | SharePoint Excel (RVTools vDisk sheet) | Same SharePoint site, RVTools vDisk sheet |
| VMware CD | SharePoint Excel (RVTools vCD sheet) | Same SharePoint site, RVTools vCD sheet |
| VMware Network | SharePoint Excel (RVTools vNetwork sheet) | Same SharePoint site, RVTools vNetwork sheet |
| VMware Snapshots | SharePoint Excel (RVTools vSnapshot sheet) | Same SharePoint site, RVTools vSnapshot sheet |
| VMware Tools | SharePoint Excel (RVTools vTools sheet) | Same SharePoint site, RVTools vTools sheet |
| Azure VMs | SharePoint **CSV** | `AzureResourceGraphFormattedResults-All%20Azure%20Virtual%20Machines.csv` |
| Azure Disks | SharePoint **CSV** | `AzureResourceGraphFormattedResults-All%20Azure%20Disks.csv` |
| Azure VM Sizes | SharePoint Excel | `AzureVMSizes.xlsx` (Sheet: "Sheet1") |
| Azure Arc Machines | SharePoint **CSV** | `AzureResourceGraphFormattedResults-All%20Servers%20in%20Azure%20Arc.csv` |
| Azure Arc SQL Instances | SharePoint **CSV** | `AzureResourceGraphFormattedResults-Azure%20Arc%20SQL%20Instances.csv` |
| Azure Arc SQL Databases | SharePoint **CSV** | `AzureResourceGraphFormattedResults-Azure%20Arc%20SQL%20Databases.csv` |
| Azure SQL Virtual Machines | SharePoint **CSV** | `AzureResourceGraphFormattedResults-Azure%20SQL%20Virtual%20Machines.csv` |
| Azure VirtualNetworks | SharePoint **CSV** | `AzureResourceGraphFormattedResults-Virtual%20Networks.csv` |
| SolarWinds VMs | SharePoint Excel | `Report_All_Virtual_Machines.xlsx` |

All files above are in the base SharePoint folder: `https://amlinonline.sharepoint.com/sites/DCExit-MigrationTeam/Shared%20Documents/Enterprise%20Apps%20DC%20Exit/Power%20BI%20Data%20Models/Discovery%20Data%20Model/`

## Asset Management & Inventory

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Active Directory Computers | SharePoint Excel | `MSA%20Report%20-%20All%20Servers.xlsx` |
| ConfigMgr All Devices | SharePoint Excel | `CM.xlsx` (Sheet: "CM") |
| ConfigMgr Encryption Status | SharePoint Excel | `Encryption%20Status.xlsx` (Sheet: "EncryptionStatus") |
| Snow Computers | SharePoint Excel | `ComputersList.xlsx` |
| Snow SQL Servers | SharePoint Excel | `StockReport.xlsx` (Sheet: "Report0") |
| Application Catalogue Working Catalogue | SharePoint Excel (parameter) | Uses `ApplicationCataloguePath` parameter — points to Apps Cat Excel on DataCentreStrategy SharePoint |
| Application Catalogue Wave Plan | SharePoint Excel (parameter) | Uses `ApplicationCataloguePath` parameter |
| Application Catalogue Discrepancies | SharePoint Excel (parameter) | Uses `ApplicationCataloguePath` parameter |
| Application Catalogue Missing From Data Model | SharePoint Excel (parameter) | Uses `ApplicationCataloguePath` parameter |
| ServiceNow CMDB | SharePoint Excel | `cmdb_ci_server.xlsx` |
| ServiceNow Decommission Change Requests | SharePoint Excel | `change_request.xlsx` (Sheet: "Page 1") |
| ServiceNow Decommission Service Requests | SharePoint Excel | `sc_item_option_mtom.xlsx` (Sheet: "Page 1") |
| ServiceNow MSSQL Databases | SharePoint Excel | `cmdb_ci_db_mssql_database.xlsx` (Sheet: "Page 1") |
| ServiceNow MSSQL Instances | SharePoint Excel | `cmdb_ci_db_mssql_database.xlsx` (Sheet: "Page 1") |
| Ops Theater RAETSMARINE Apps | **Web scraping** (internal HTML) | `http://spdoca0001.raetsmarine.local/servers_func.html` — scrapes HTML table with `id='servers_srt'` |

## Monitoring Tools

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Qualys Assets | SharePoint **CSV** | `AI_Asset_List_amncr-vs1.csv` |
| Symantec Endpoint Protection Devices | SharePoint **CSV** | `SEP_Device_status.csv` |
| QRadar Log Sources | SharePoint Excel | `QRadar_All_Devices.xlsx` (Sheet: "QRadar Logsources") |
| Panaseer All Devices | SharePoint **CSV** | `Panaseer%20Export%20-%20prod_inv_msr_n-device.csv` |
| SolarWinds Nodes | SharePoint Excel | `Report_All_Nodes_(Servers).xlsx` |

## Mapping Tables

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Mapping Application to Application Catalogue | SharePoint Excel | `Server%20&%20Application%20Mapping.xlsx` (ApplicationCatalogueMapping table) |
| Mapping Diamond IP IPControl Subnet to Location | **Embedded JSON** | Hardcoded lookup data (Binary.Decompress/Base64 in M query) |
| Mapping DomainFQDN to DomainNetBIOS | **Embedded JSON** | Hardcoded lookup data (Binary.Decompress/Base64) |
| Mapping DomainNetBIOS to DomainFQDN | **Embedded JSON** | Hardcoded lookup data (Binary.Decompress/Base64) |
| Mapping Location | **Embedded JSON** | Hardcoded location normalisation data (Binary.Decompress/Base64) |
| Mapping Manual Missing Subnets | **Embedded JSON** | Hardcoded subnet overrides (Binary.Decompress/Base64), Source = "Manual" |
| Mapping Manual Server To Location | **Embedded JSON** | Hardcoded server-location overrides (Binary.Decompress/Base64) |
| Mapping Server to Application | SharePoint Excel | `Server%20&%20Application%20Mapping.xlsx` (different folder: `.../Enterprise%20Apps%20DC%20Exit/`) |
| Mapping VLAN to Application | **Internal reference** | References `VMware VMs` table (not an external source) |
| Mapping VM to ServerName | SharePoint Excel | `https://amlinonline.sharepoint.com/.../Mapping-VM-ServerName.xlsx` |

## Merged Tables

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Merged IP Addresses | Power Query (M) merge | Combines IP data from multiple source tables |
| Merged Subnets | Power Query (M) merge | Combines subnet data from source tables |
| Merged Subnets Expanded | Power Query (M) + custom function | Expands subnets to individual IPs using `ExpandSubnet` function |
| Merged SQL Databases | Power Query (M) merge | Combines SQL database records from Azure Arc, ServiceNow, DB Inventory |
| Merged SQL Instances | Power Query (M) merge | Combines SQL instance records from multiple sources |

## Network

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Diamond IP IPControl Subnets | SharePoint **CSV** | `dhcpServerAssignmentReport.csv` |
| Network Team Global IP Address Allocation | SharePoint Excel | `Global%20IP%20Address%20Allocation.xlsx` (Sheet: "New Overview") |
| Network Team Network Switch Traffic | SharePoint Excel | `vlan-switch-export.xlsx` (Sheet: "Switch Connected Devices") — in different folder: `.../Networking/vlan-analysis/` |

## Other Teams

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| Citrix Team Published Applications | SharePoint Excel | `Published-app-And-Server.xlsx` (Sheet: "Sheet1") |
| DR Manager Important Business Services | SharePoint Excel | `IBS%20Master%20List%20v0.5.xlsx` (Sheet: "IBS Master List") |
| Environments Team Environment Management Database | SharePoint Excel | `data.xlsx` (Sheet: "Export") |

## DB Inventory

| Table | Source Type | Source URL/Connection |
|-------|------------|----------------------|
| DB Inventory System Databases | SharePoint **CSV** | `DBInventorySystem-Export.csv` |
| DB Inventory System Instances | SharePoint **CSV** | `DBInventorySystem-Export.csv` (same file, different transform) |

## Calculated Tables (DAX — no external source)

| Table | Source |
|-------|--------|
| _All Devices | DAX calculated table (UNION + EXCEPT pattern) |
| _All Projects Scope | DAX calculated table (GENERATE pattern) |
| _All Applications by Server | DAX calculated table (GENERATE pattern) |

## Diagnostics Queries (Developer-Only)

These are **not production data sources**. They reference local diagnostic JSON files from a specific developer's machine (`C:\Users\GDVSHB\...`).

| Expression | Source |
|------------|--------|
| `Qualys Assets_Source_Detailed_2025-09-19_17:11` | Local JSON file — Power BI diagnostics trace |
| `Qualys Assets_Source_Aggregated_2025-09-19_17:11` | Local JSON file — Power BI diagnostics trace |
| `Qualys Assets_Source_Partitions_2025-09-19_17:11` | Local JSON file — Power BI diagnostics trace |

> **[REVIEW]**: These diagnostics queries reference paths on the `GDVSHB` user profile (`C:\Users\GDVSHB\AppData\Local\Microsoft\Power BI Desktop\Traces\Diagnostics\`). They are leftover from a previous developer's debugging session and will fail on other machines. Consider removing them if no longer needed.
