# Module 06: Mapping & Support Tables

## What You'll Learn
The 10 mapping tables, 6 merged tables, network tables, and 3 M functions.

### Sections:

**1. Why Mapping Tables Exist**
Source systems don't use consistent naming. One says "Romford", another says "ROM", another uses a subnet range. Mapping tables normalize these differences so the model can compare data across systems.

**2. Location Mapping (5 tables)**
Location is one of the most complex aspects of the model. Five different sources provide location data, and _All Devices uses a 5-tier fallback chain (Module 10 covers the full DAX):

1. `Mapping Location` -- Primary lookup table. Uses embedded Base64 JSON data (no external file dependency). Maps server attributes to standard location names.
2. `Mapping Diamond IP IPControl Subnet to Location` -- Maps Diamond IP IPAM subnets to locations. Column: Container, Location.
3. `Mapping Manual Server To Location` -- Manual overrides for specific servers. Columns: Name, Location. The last resort in the fallback chain.
4. `Mapping Manual Missing Subnets` -- Manual subnet-to-location for subnets not in Diamond IP. Columns: Location, Subnet.
5. `Merged Subnets` -- Combined subnet data with location. Used in the fallback: if we know the server's IP Subnet, look up its location here.

The 5-tier fallback for Location in `_All Devices` is:
1. VMware VMs Location (from vCenter data)
2. Azure VMs Location (from Azure)
3. VMware Hosts Location (for ESXi hosts)
4. Merged Subnets Location (via IP Subnet)
5. Mapping Manual Server To Location (manual override, last resort)

**3. Domain Mapping (2 tables)**
Active Directory domains have two name formats:
- **FQDN**: `raetsmarine.local` (Fully Qualified Domain Name)
- **NetBIOS**: `RAETSMARINE` (short name)

Two bidirectional mapping tables handle this:
- `Mapping DomainFQDN to DomainNetBIOS` -- FQDN to NetBIOS lookup. Columns: FQDN, NetBIOS.
- `Mapping DomainNetBIOS to DomainFQDN` -- NetBIOS to FQDN lookup. Columns: NetBIOS, FQDN.

These two tables have a relationship between them (the only bidirectional relationship in the model): connected via the NetBIOS column.

**4. Application & VM Mapping (3 tables)**
- `Mapping Server to Application` -- The critical many-to-many mapping. Columns: Server, Application. One server can run multiple applications; one application can span multiple servers. Used by `_All Devices` (Application column via CONCATENATEX), `_All Projects Scope`, and `_All Applications by Server`.
- `Mapping Application to Application Catalogue` -- Normalizes application names from the mapping table to official Application Catalogue names. Columns: Name, Name (Application Catalogue), Reference, KS Comments.
- `Mapping VM to ServerName` -- Maps VMware VM names to server names when they differ. Column: VM, ServerName. Has a relationship to `VMware VMs`.

**5. VLAN Mapping (1 table)**
- `Mapping VLAN to Application` -- Maps VLAN IDs to applications. Columns: VLAN, Application.

**6. Merged Tables (6)**
Merged tables combine data from multiple sources into unified views:

- `Merged IP Addresses` -- Combines IP addresses from AD, ConfigMgr, Snow, and other sources. Columns: Name, IP Address, IP Address (Decimal). Used by `_All Devices` for the IP Address and IP Addresses calculated columns.
- `Merged Subnets` -- Combines subnet data with location information. Columns: Subnet, Location. Used in the Location fallback chain.
- `Merged Subnets Expanded` -- Expands every subnet into individual IP addresses using the `ExpandSubnet` M function. Columns: IP Address, IP Address (Decimal), Subnet. Used for IP Subnet lookup in `_All Devices`. Has an inactive relationship to Merged IP Addresses (to avoid circular filtering).
- `Merged SQL Databases` -- Combines SQL database records from Azure Arc, ServiceNow, DB Inventory, and Snow. Columns include Full Instance Name, Database Name.
- `Merged SQL Instances` -- Combines SQL instance records from multiple sources. Column: Full Instance Name.
- `SolarWinds VMs` -- Discovered VMs from SolarWinds (also listed with Hosting tables). Columns: Name, Virtualization Platform.

**7. Network Tables (4)**
- `Diamond IP IPControl Subnets` -- IPAM subnet inventory. Columns: Container, Subnet, Description.
- `Network Team Global IP Address Allocation` -- The network team's master subnet allocation table. Column: Subnet. This is the "hub" table in the network relationship cluster -- 5 tables connect TO it via Subnet.
- `Network Team Network Switch Traffic` -- Switch port traffic data. Columns: IP Address, VLAN. Used by `_All Devices` Network column.
- `Azure VirtualNetworks` -- Azure vNet and subnet definitions. Columns: name, subnetAddressPrefix. Used by `_All Devices` Azure vNet column.

**8. The 3 Custom M Functions**
Defined in `expressions.tmdl` in the Functions query group:

`ExpandSubnet` -- Takes a CIDR subnet (e.g., "10.0.0.0/24") and returns a list of all IP addresses in that range. Used by `Merged Subnets Expanded` to create a row for every IP in every subnet. The function:
1. Splits the CIDR notation into base IP and prefix length
2. Converts the base IP to an integer
3. Calculates total IPs (2^(32-prefix))
4. Generates all IP integers
5. Converts back to dotted-decimal notation

`ConvertToMACAddress` -- Converts VMware-format MAC addresses (e.g., "0050.56a2.1234") to standard colon-separated format ("00:50:56:a2:12:34"). Splits on dots, recombines, and inserts colons every 2 characters.

`ConvertIPToDecimal` -- Converts a dotted-decimal IP address to its decimal integer equivalent. Used for IP address range matching. Formula: (Part1 * 256^3) + (Part2 * 256^2) + (Part3 * 256) + Part4.

**9. Hands-On Exercise**
1. Open `expressions.tmdl` in a text editor -- find the three functions and read their M code
2. In Power Query Editor, find the Functions group -- click on `ExpandSubnet` to see its definition
3. Find `Merged Subnets Expanded` -- see how it uses ExpandSubnet
4. Find `Mapping Location` -- look for the Base64 embedded data
5. Find `Mapping Server to Application` -- note how one server appears multiple times (many-to-many)

**10. Key Takeaways**
- Mapping tables normalize inconsistent naming across source systems
- Location has a 5-tier fallback chain (VMware, Azure, ESXi Host, Subnet, Manual)
- Merged tables combine data from multiple sources into unified views
- Three M functions handle subnet expansion, MAC formatting, and IP conversion
- `Mapping Server to Application` is the key many-to-many table linking servers to applications
