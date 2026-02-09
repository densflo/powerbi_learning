# Module 04: Hosting Platform Tables

## What You'll Learn
The 10+ tables that capture where servers live -- VMware, Azure, SolarWinds.

---

## 1. VMware Tables (7)

The VMware cluster is the largest group. Data comes from RVTools -- a tool that exports vCenter inventory to Excel.

Three vCenters feed data:

- **SVCENT0005** -- Primary on-premises vCenter
- **AVSUKSouth** -- Azure VMware Solution UK South
- **AVSUKWest** -- Azure VMware Solution UK West

### Tables

**`VMware VMs`** -- The main VM inventory. Combines all 3 vCenters using `Table.Combine()`. Key columns: Name, VM, Host, Cluster, Powerstate (poweredOn/poweredOff), CPUs, Memory, Primary IP Address, Network, OS, Location. This is the foundation of `_All Devices`.

**`VMware Hosts`** -- Physical ESXi host servers. Key columns: Name, Host (FQDN), Cluster, CPU Model, CPU Count, Cores, Memory, Location. Has relationships TO it from 6 other VMware tables.

**`VMware Disks`** -- Virtual disk details. Key columns: VM, Host, Capacity MiB, Disk Path. Used by `_All Devices` to calculate total Disk Capacity.

**`VMware Network`** -- Network adapter details. Key columns: VM, Host, Network, Adapter (Vmxnet3 vs E1000). Used to check for E1000 adapters (legacy, should be migrated).

**`VMware Snapshots`** -- Snapshot inventory. Key columns: VM, Host, Name, Date, Size. Used to flag which VMs have snapshots.

**`VMware Tools`** -- VMware Tools version per VM. Key columns: VM, Host, Tools Version. Used for Tools Version column in `_All Devices`.

**`VMware CD`** -- Mounted CD/DVD media. Key columns: VM, Host, ISO Path. Used to flag which VMs have mounted media (should be cleaned up before migration).

### The VM vs Name Distinction

This distinction is critical throughout the model:

- **VM** is the VMware display name (often the FQDN like `server01.domain.com`)
- **Name** is the short hostname (`server01`)

Many lookups in `_All Devices` match on Name, not VM. If a server's VMware name does not follow the FQDN convention, matching can break. Keep this in mind when troubleshooting missing data.

---

## 2. Azure Tables (3)

**`Azure VMs`** -- Azure virtual machines. Key columns: Name, IPAddress, PowerState (VM running/VM deallocated), Location, Cores, Memory. These enter `_All Devices` directly in the base UNION.

**`Azure Disks`** -- Azure managed disks. Key columns: managedBy (server name), diskSizeGB. Used for Disk Capacity calculation.

**`Azure VM Sizes`** -- Reference table of Azure VM SKU sizes. Used for reference, not directly in calculated columns.

---

## 3. SolarWinds VMs (1)

**`SolarWinds VMs`** -- Discovered VMs from SolarWinds, specifically Hyper-V VMs. Key columns: Name, Virtualization Platform. Filtered to `Virtualization Platform = "Hyper-V"` when added to `_All Devices`. These are the 3rd priority in the UNION chain (after VMware and Azure).

---

## 4. How These Feed Into _All Devices

The hosting platform tables enter `_All Devices` through a priority-based UNION chain. Each step uses `EXCEPT` to remove servers that were already added by a higher-priority source:

1. **VMware VMs + VMware Hosts + Azure VMs** (base UNION) -- These are combined first and form the foundation. Every server found in VMware or Azure is included.
2. **SolarWinds VMs filtered to Hyper-V** (EXCEPT to remove duplicates) -- Only Hyper-V VMs that were NOT already in the VMware/Azure set are added.
3. **Snow Computers filtered to non-hypervisor** (EXCEPT) -- Physical servers from Snow that were not already captured.
4. **SolarWinds Nodes filtered to server-type** (EXCEPT) -- Remaining network-monitored servers not found in any prior source.

This hierarchy means that if a server appears in both VMware and SolarWinds, the VMware record wins. The EXCEPT deduplication prevents double-counting.

---

## 5. Key Columns Across Hosting Tables

Common columns and what they mean:

| Column | Meaning | Example |
|--------|---------|---------|
| **Name** | Short hostname | `SVWEB0001` |
| **VM** | Full VMware VM name (may include domain) | `SVWEB0001.domain.com` |
| **Host** | The physical host server running the VM | `SVESXI0012.domain.com` |
| **Cluster** | The vCenter cluster grouping | `Production-Cluster-01` |
| **Powerstate / PowerState** | Is the server ON or OFF? | `poweredOn` / `VM running` |
| **Location** | Data center location | `Romford`, `Hemel Hempstead`, `AVS - UK South` |

Note the casing difference: VMware uses `Powerstate` (lowercase s), Azure uses `PowerState` (uppercase S). The `_All Devices` DAX normalizes these into a single column.

---

## 6. Hands-On Exercise

1. In Power BI, go to **Data view**.
2. Click on **`VMware VMs`** -- note the row count and key columns.
3. Look at the **Powerstate** column -- see `poweredOn` vs `poweredOff`.
4. Click on **`VMware Hosts`** -- note how many physical hosts exist.
5. Click on **`Azure VMs`** -- compare the column names to VMware VMs. Notice the differences in naming conventions (e.g., `PowerState` vs `Powerstate`).
6. In **Model view**, find the VMware cluster -- see how Disks, Network, Snapshots, Tools, and CD all connect to VMware Hosts via the Host column.
7. Try to find a server that appears in both `VMware VMs` and `Azure VMs` -- this helps you understand why the EXCEPT deduplication is necessary.

---

## 7. Key Takeaways

- VMware is the largest data source (7 tables from 3 vCenters).
- VM vs Name: VM is the full VMware name, Name is the short hostname. Most lookups use Name.
- Azure and SolarWinds provide additional hosting platforms beyond VMware.
- Priority in `_All Devices`: VMware > Azure > Hyper-V > Snow Physical > SolarWinds Nodes.
- The EXCEPT pattern ensures each server appears only once, from its highest-priority source.
- VMware detail tables (Disks, Network, Snapshots, Tools, CD) connect to VMware Hosts via the Host column, forming a star-like relationship cluster.
