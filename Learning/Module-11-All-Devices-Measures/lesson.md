# Module 11: _All Devices -- Measures & Gap Analysis

## What You'll Learn
The 10 measures, the gap analysis pattern, and how measures differ from calculated columns.

### Sections:

**1. Measures vs Calculated Columns -- A Deeper Look**
In Module 08, we introduced the concept. Now let's see it in practice:

**Calculated column** (evaluated per row, stored in model):
- `In Qualys = IF(NOT(ISBLANK(LOOKUPVALUE(...))), TRUE, FALSE)`
- This TRUE/FALSE value is computed once per row and saved
- Doesn't change when you filter the report

**Measure** (evaluated dynamically, responds to filters):
- `Not In Qualys = CALCULATE(COUNTA([In Qualys]), [In Qualys] IN { FALSE })`
- This count changes depending on slicers, filters, and visual context
- Shows "50" when viewing all servers, but "12" when filtered to Location = "Romford"

Together, they form a pattern: the column flags each row, the measure counts the flags. The column provides granularity; the measure provides aggregation.

**2. The Count Measure**
```dax
measure 'No. of Devices' = COUNTA([Name])
```
Simple count of all non-blank Name values in the current filter context. COUNTA (Count All) counts non-blank values, including text and numbers. This is the primary KPI -- "how many servers are we looking at?"

**3. Boolean Count Measures (3)**
These count how many servers have specific characteristics:

```dax
measure 'Has Mounted Media' =
    VAR HasMountedMediaCount =
    CALCULATE(
        COUNTA('_All Devices'[Mounted Media]),
        '_All Devices'[Mounted Media] IN { TRUE }
    )
    RETURN
    IF(
        HasMountedMediaCount = BLANK(),
        0,
        HasMountedMediaCount
    )
```

All three follow the same pattern:
| Measure | Column Checked | Condition |
|---------|---------------|-----------|
| Has Mounted Media | Mounted Media | = TRUE |
| Has Encrypted Drives | Encrypted Drives | = TRUE |
| Has Snapshots | Snapshots | = TRUE |

Pattern breakdown:
1. CALCULATE modifies filter context to only count TRUE values
2. COUNTA counts the non-blank values in that filtered context
3. IF wraps the result: if BLANK (no matches), return 0 instead
4. The VAR...RETURN pattern stores the intermediate result for the BLANK check

**4. Gap Analysis Measures (5)**
The most business-critical measures. They answer: "How many servers are NOT monitored by each tool?"

```dax
measure 'Not In Qualys' =
    VAR NotInQualysCount =
    CALCULATE(
        COUNTA('_All Devices'[In Qualys]),
        '_All Devices'[In Qualys] IN { FALSE }
    )
    RETURN
    IF(
        NotInQualysCount = BLANK(),
        0,
        NotInQualysCount
    )
```

All five gap analysis measures:
| Measure | Boolean Column | Business Question |
|---------|---------------|-------------------|
| Not In Qualys | In Qualys | How many servers lack vulnerability scanning? |
| Not In SEP | In SEP | How many servers lack antivirus? |
| Not In CMDB | In CMDB | How many servers aren't in the CMDB? |
| Not In SWO Nodes | In SolarWinds Nodes | How many servers lack network monitoring? |
| Not In SWO VMs | In SolarWinds VMs | How many servers aren't discovered as VMs? |

These are critical for IT governance: every server SHOULD be monitored by every tool. These measures highlight the gaps.

**5. Distinct Count Measure**
```dax
measure 'No. of Networks' =
    VAR NoOfNetworks =
        DISTINCTCOUNTNOBLANK('_All Devices'[Network])
    RETURN
    IF(
        NoOfNetworks = BLANK(),
        0,
        NoOfNetworks
    )
```
DISTINCTCOUNTNOBLANK counts unique non-blank Network values. Used to show how many different networks are represented in the current filter context.

**6. The Defensive Coding Pattern**
Every measure in this model wraps its result in:
```dax
IF(result = BLANK(), 0, result)
```
Why? In Power BI:
- If no rows match a filter, measures return BLANK (not zero)
- BLANK causes cards/visuals to show "(Blank)" or disappear entirely
- Wrapping with IF ensures a clean "0" is displayed
- This is defensive coding -- handling edge cases gracefully

**7. How Measures Interact with Report Visuals**
When a measure is placed on a report visual:
- **Card visual**: Shows the single aggregated value (e.g., "Not In Qualys: 47")
- **Table visual**: Shows the measure value per row grouping
- **Slicer interaction**: When you select Location = "Romford" in a slicer, ALL measures recalculate for only Romford servers

This is the power of measures vs columns: measures automatically respond to every filter change without any additional code.

**8. Hands-On Exercise**
1. In Power BI Desktop, go to Report view
2. Find the Server Inventory page
3. Look for KPI cards showing numbers (these are measures)
4. Click on a slicer value (like a specific Location) -- watch the numbers change
5. In Data view, click on `_All Devices`
6. In the Fields pane (right side), find the measures section (they have a calculator icon)
7. Click on `Not In Qualys` -- see the formula in the formula bar
8. Create a new page, add a Card visual, drag `Not In Qualys` onto it -- see the total
9. Add a slicer for Location -- filter and watch the card update

**9. The Complete _All Devices Summary**
After Modules 09-11, you now understand:
- The UNION chain (how rows are built) -- Module 09
- The calculated columns (how rows are enriched) -- Module 10
- The measures (how rows are aggregated) -- Module 11

Together, `_All Devices` provides:
- A deduplicated list of all servers (Name + VM)
- 9 boolean flags showing presence in each source system
- 5 boolean characteristics (encryption, snapshots, media, adapter, AVS match)
- 6 COALESCE fallback columns (Domain, OS, Manufacturer, Model, Cores, Memory)
- 10+ complex enrichment columns (Device Type, Application, Location, etc.)
- 10 measures for counting and gap analysis

**10. Key Takeaways**
- Measures are dynamic (respond to filters); columns are static (per row)
- Gap analysis pattern: boolean column (per row) + count measure (aggregate)
- 5 "Not In X" measures are the business-critical gap analysis KPIs
- Defensive coding: every measure wraps with IF(BLANK(), 0, result)
- Measures power the dashboard -- they're what users see on report pages
