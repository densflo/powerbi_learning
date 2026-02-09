# TRoI Power BI Learning Course

## Overview

A progressive, self-paced course to learn Power BI through the **True Reflection of Inventory (TRoI)** model -- an enterprise IT infrastructure inventory that consolidates data from 15+ source systems (VMware, Azure, Active Directory, ServiceNow CMDB, Snow Software, SolarWinds, Qualys, and more) into a unified server and device view.

Each module builds on the previous one, starting with Power BI fundamentals and ending with real-world maintenance scenarios. By the end of the course, you will understand the full TRoI architecture: its Power Query imports, DAX calculated tables, relationship model, report pages, and the patterns that hold them together.

---

## Prerequisites

- **Power BI Desktop** installed on your machine.
- **Access to the TRoI `.pbip` project file** (`True Reflection of Inventory (TRoI).pbip` in the repository root).
- Basic familiarity with spreadsheets and data concepts is helpful but not required.

---

## How to Use This Course

1. **Open each module's `lesson.md` in order**, starting with Module 01. The modules are designed to be sequential -- later modules reference concepts introduced in earlier ones.
2. **Complete the hands-on exercises** included in each lesson. These exercises use the live TRoI model in Power BI Desktop, so keep it open alongside the lesson material.
3. **Track your progress** in each module's `PROGRESS.md` file. Use it to mark completed sections, record notes, and flag questions for review.

---

## Estimated Total Time

**Approximately 14 hours** (~1 hour per module). Some modules may take slightly longer depending on your prior experience with Power BI and DAX.

---

## Master Progress Checklist

Use this checklist to track your overall progress through the course.

- [ ] **Module 01: Getting Started** -- Power BI basics, the PBIP format, and the three-layer architecture (Power Query, DAX, Report)
- [ ] **Module 02: The Model Map** -- All 61 tables at a glance, organized by query group
- [ ] **Module 03: Reading Source Tables** -- TMDL file anatomy and Power Query (M) basics
- [ ] **Module 04: Hosting Platform Tables** -- VMware, Azure, and SolarWinds hosting data
- [ ] **Module 05: Asset and Monitoring Tables** -- Active Directory, ConfigMgr, Snow, ServiceNow, Qualys, SEP
- [ ] **Module 06: Mapping and Support Tables** -- Location fallback logic, domain mapping, custom M functions
- [ ] **Module 07: Relationships** -- All 22 relationships, the 5 relationship clusters, and why `_All Devices` has none
- [ ] **Module 08: DAX Fundamentals** -- Key functions: LOOKUPVALUE, COALESCE, UNION, EXCEPT
- [ ] **Module 09: _All Devices -- The UNION Chain** -- The 7-stage deduplication engine that builds the core table
- [ ] **Module 10: _All Devices -- Calculated Columns** -- The 30+ columns that enrich each device record
- [ ] **Module 11: _All Devices -- Measures and Gap Analysis** -- 10 measures and the gap analysis pattern
- [ ] **Module 12: Projects Scope and Applications** -- The other 2 calculated tables built with GENERATE
- [ ] **Module 13: Report Pages** -- All 8 report pages, visuals, slicers, and cross-filtering behavior
- [ ] **Module 14: Maintenance and Gotchas** -- Extending the model, known issues, and quick reference

---

## Tips for Success

- **Read modules in order.** Each module builds on concepts introduced in the previous one. Skipping ahead may leave gaps in understanding.
- **Use Power BI Desktop alongside each lesson.** The exercises are designed to be worked through with the TRoI model open. Reading alone is not a substitute for hands-on exploration.
- **Write notes in each module's `PROGRESS.md`.** Capture what you learned, what surprised you, and any questions that come up. These notes will be valuable when you revisit the material later.
- **Take breaks between modules.** Each module is roughly one hour. Spreading the course over multiple days helps with retention.
- **Refer back to earlier modules as needed.** Module 02 (The Model Map) and Module 08 (DAX Fundamentals) are especially useful as ongoing references.

---

## Folder Structure

```
Learning/
    README.md                                    <-- You are here
    Module-01-Getting-Started/
    Module-02-The-Model-Map/
    Module-03-Reading-Source-Tables/
    Module-04-Hosting-Platform-Tables/
    Module-05-Asset-and-Monitoring-Tables/
    Module-06-Mapping-and-Support-Tables/
    Module-07-Relationships/
    Module-08-DAX-Fundamentals/
    Module-09-All-Devices-The-UNION-Chain/
    Module-10-All-Devices-Calculated-Columns/
    Module-11-All-Devices-Measures/
    Module-12-Projects-Scope-and-Applications/
    Module-13-Report-Pages/
    Module-14-Maintenance-and-Gotchas/
```

Each module folder contains a `lesson.md` with the teaching material and a `PROGRESS.md` for tracking your completion and notes.
