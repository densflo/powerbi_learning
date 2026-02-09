# 07 — Power Query Functions

The TRoI model contains 3 custom Power Query (M) functions in the `Child Queries\Functions` query group.

---

## ExpandSubnet

**Signature**: `ExpandSubnet(Subnet as text) => list`

**Purpose**: Converts a CIDR notation subnet (e.g., `"10.0.0.0/24"`) into a list of all individual IP addresses within that subnet.

**Used By**: `Merged Subnets Expanded` table — to create a row for every IP address in every known subnet, enabling IP-to-subnet lookups.

### Logic

```
ExpandSubnet = (Subnet as text) =>
let
    // 1. Split CIDR notation into base IP and prefix length
    SubnetParts = Text.Split(Subnet, "/"),
    BaseIP = SubnetParts{0},
    PrefixLength = Number.From(SubnetParts{1}),

    // 2. Convert base IP to a single integer
    BaseIPParts = List.Transform(Text.Split(BaseIP, "."), Number.From),
    BaseIPInteger = (Part1 * 256^3) + (Part2 * 256^2) + (Part3 * 256) + Part4,

    // 3. Calculate total IPs in the subnet (2^(32 - prefix))
    TotalIPs = Number.Power(2, 32 - PrefixLength),

    // 4. Generate list of all IP integers
    IPIntegers = List.Transform({0..TotalIPs - 1}, each BaseIPInteger + _),

    // 5. Convert each integer back to dotted-decimal notation
    IPAddresses = List.Transform(IPIntegers, each
        Text.From(IntDiv(_, 256^3)) & "." &
        Text.From(IntDiv(Mod(_, 256^3), 256^2)) & "." &
        Text.From(IntDiv(Mod(_, 256^2), 256)) & "." &
        Text.From(Mod(_, 256))
    )
in
    IPAddresses
```

### Example
- Input: `"10.1.2.0/24"`
- Output: List of 256 IPs from `"10.1.2.0"` to `"10.1.2.255"`

### Performance Note
Large subnets (e.g., /16 = 65,536 IPs, /8 = 16.7M IPs) will generate very large lists. The model should only be expanding subnets that are reasonably sized.

---

## ConvertToMACAddress

**Signature**: `ConvertToMACAddress(MACAddress as text) => text`

**Purpose**: Converts a dot-separated MAC address (Cisco format) to colon-separated format.

### Logic

```
ConvertToMACAddress = (MACAddress as text) as text =>
let
    // 1. Split by dots (e.g., "0050.5689.1234" => {"0050", "5689", "1234"})
    SplitText = Text.Split(MACAddress, "."),

    // 2. Combine into one string (e.g., "005056891234")
    CombinedText = Text.Combine(SplitText, ""),

    // 3. Insert colons every 2 characters
    FormattedMACAddress =
        Text.Middle(CombinedText, 0, 2) & ":" &
        Text.Middle(CombinedText, 2, 2) & ":" &
        Text.Middle(CombinedText, 4, 2) & ":" &
        Text.Middle(CombinedText, 6, 2) & ":" &
        Text.Middle(CombinedText, 8, 2) & ":" &
        Text.Middle(CombinedText, 10, 2)
in
    FormattedMACAddress
```

### Example
- Input: `"0050.5689.1234"`
- Output: `"00:50:56:89:12:34"`

---

## ConvertIPToDecimal

**Signature**: `ConvertIPToDecimal(IPAddress as text) => number`

**Purpose**: Converts a dotted-decimal IP address to its decimal (integer) equivalent.

**Used By**: `Merged Subnets Expanded` and `Merged IP Addresses` tables — to create the `IP Address (Decimal)` column, enabling numeric range comparisons between IPs and subnet ranges.

### Logic

```
ConvertIPToDecimal = (IPAddress as text) as number =>
let
    IP = Text.Split(IPAddress, "."),
    Part1 = Number.FromText(IP{0}),
    Part2 = Number.FromText(IP{1}),
    Part3 = Number.FromText(IP{2}),
    Part4 = Number.FromText(IP{3})
in
    (Part1 * 256 * 256 * 256) + (Part2 * 256 * 256) + (Part3 * 256) + Part4
```

### Example
- Input: `"10.1.2.3"`
- Output: `167837187` (10×16777216 + 1×65536 + 2×256 + 3)
