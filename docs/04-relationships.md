# 04 — Relationships

## Summary

The TRoI model has **22 relationships** (21 active, 1 inactive). All use single-column joins. The relationships form several distinct clusters.

---

## Complete Relationship Table

| # | From Table | From Column | To Table | To Column | Active | Auto-Detected |
|---|-----------|-------------|----------|-----------|--------|---------------|
| 1 | Mapping DomainFQDN to DomainNetBIOS | NetBIOS | Mapping DomainNetBIOS to DomainFQDN | NetBIOS | Yes | No |
| 2 | VMware VMs | Host | VMware Hosts | Host | Yes | No |
| 3 | VMware Disks | Host | VMware Hosts | Host | Yes | No |
| 4 | VMware Snapshots | Host | VMware Hosts | Host | Yes | No |
| 5 | VMware CD | Host | VMware Hosts | Host | Yes | No |
| 6 | VMware Tools | Host | VMware Hosts | Host | Yes | No |
| 7 | VMware Network | Host | VMware Hosts | Host | Yes | No |
| 8 | Application Catalogue Working Catalogue | Application Reference | Application Catalogue Wave Plan | Application Reference | Yes | No |
| 9 | Diamond IP IPControl Subnets | Container | Mapping Diamond IP IPControl Subnet to Location | Container | Yes | Yes |
| 10 | Merged Subnets | Subnet | Network Team Global IP Address Allocation | Subnet | Yes | Yes |
| 11 | Mapping Manual Missing Subnets | Location | Mapping Diamond IP IPControl Subnet to Location | Location | Yes | Yes |
| 12 | Merged Subnets Expanded | Subnet | Network Team Global IP Address Allocation | Subnet | Yes | Yes |
| 13 | Merged IP Addresses | IP Address (Decimal) | Merged Subnets Expanded | IP Address (Decimal) | **No** | Yes |
| 14 | Merged IP Addresses | Subnet | Network Team Global IP Address Allocation | Subnet | Yes | Yes |
| 15 | ServiceNow MSSQL Databases | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 16 | Azure Arc SQL Databases | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 17 | DB Inventory System Databases | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 18 | Merged SQL Databases | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 19 | Merged SQL Instances | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 20 | DB Inventory System Instances | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 21 | ServiceNow MSSQL Instances | Full Instance Name | Azure Arc SQL Instances | Full Instance Name | Yes | Yes |
| 22 | Mapping VM to ServerName | VM | VMware VMs | VM | Yes | No |

---

## Relationship Clusters

### VMware Cluster (6 relationships)

All VMware detail tables relate to `VMware Hosts` via the `Host` column:

```
VMware VMs ──────────┐
VMware Disks ────────┤
VMware Snapshots ────┤── Host ──▶ VMware Hosts [Host]
VMware CD ───────────┤
VMware Tools ────────┤
VMware Network ──────┘
```

Additionally, `Mapping VM to ServerName` relates to `VMware VMs` via `VM`:

```
Mapping VM to ServerName ── VM ──▶ VMware VMs [VM]
```

### SQL Instance Cluster (7 relationships)

All SQL-related tables relate to `Azure Arc SQL Instances` as the central hub via `Full Instance Name`:

```
ServiceNow MSSQL Databases ──────┐
Azure Arc SQL Databases ─────────┤
DB Inventory System Databases ───┤── Full Instance Name ──▶ Azure Arc SQL Instances
Merged SQL Databases ────────────┤
Merged SQL Instances ────────────┤
DB Inventory System Instances ───┤
ServiceNow MSSQL Instances ──────┘
```

### Network / Subnet Cluster (4 relationships)

```
Merged Subnets ──────────── Subnet ──▶ Network Team Global IP Address Allocation [Subnet]
Merged Subnets Expanded ─── Subnet ──▶ Network Team Global IP Address Allocation [Subnet]
Merged IP Addresses ─────── Subnet ──▶ Network Team Global IP Address Allocation [Subnet]

Merged IP Addresses ── IP Address (Decimal) ──▶ Merged Subnets Expanded [IP Address (Decimal)]
                       ↑ INACTIVE
```

> **Note**: The IP Address (Decimal) relationship is **inactive**. This means it's available for use via `USERELATIONSHIP()` in DAX but doesn't filter automatically.

### Diamond IP / Location Cluster (2 relationships)

```
Diamond IP IPControl Subnets ── Container ──▶ Mapping Diamond IP IPControl Subnet to Location [Container]
Mapping Manual Missing Subnets ── Location ──▶ Mapping Diamond IP IPControl Subnet to Location [Location]
```

### Domain Mapping (1 relationship)

```
Mapping DomainFQDN to DomainNetBIOS ── NetBIOS ──▶ Mapping DomainNetBIOS to DomainFQDN [NetBIOS]
```

### Application Catalogue (1 relationship)

```
Application Catalogue Working Catalogue ── Application Reference ──▶ Application Catalogue Wave Plan [Application Reference]
```

---

## Design Notes

- **No relationships to `_All Devices`**: The calculated table `_All Devices` does not participate in any model relationships. All cross-table lookups from `_All Devices` are done via `LOOKUPVALUE()` in DAX calculated columns rather than through model relationships.
- **Star schema hub**: `Azure Arc SQL Instances` and `VMware Hosts` serve as hub/dimension tables with multiple fact tables pointing to them.
- **Auto-detected relationships**: 13 of the 22 relationships were auto-detected by Power BI (prefixed with `AutoDetected_` in the TMDL).
