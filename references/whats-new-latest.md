# What's New — Latest SAC Release

**Source**: SAP Help — What's New in SAP Analytics Cloud Planning
**Last updated**: 2026-10-18
**SAC version**: 2026.15 (Q3 2026 — most recent as of this knowledge base update)

---

## How to Keep This File Current

This file should be updated within **2 weeks of each SAC QRC release**.

SAC releases quarterly (QRC = Quarterly Release Cycle). Update process:
1. Visit [SAP Help: What's New in SAC](https://help.sap.com/whats-new/cf0cb2cb149647329b5d02aa96303f56)
2. Filter by "Planning" and "Data Actions" categories
3. Extract relevant changes into the sections below
4. Update the version header above
5. Commit with message: `Update whats-new for SAC [version]`

---

## SAC 2026.15 — Data Actions and Planning Changes

**Source used**: SAP Help Portal Q3 2026 custom export (8,628 lines). Reference files in this knowledge base were updated from this source on 2026-10-18.

### Key changes from 2026.15 source confirmed in reference files
- Full Advanced Formula function reference updated (see `advanced-formula-rules.md`)
- Data Action Tracing documentation added (see `data-action-tracing.md`)
- Performance best practices documented (see `performance-tips.md`)
- All allocation, multi-action, and context/scope references verified against 2026.15 content

---

## SAC 2025.19 — Data Actions and Planning Changes

### New and Enhanced Features

**Advanced Formula — Improved error messages**  
Error messages in the execution log now include the formula line number and column, making it easier to locate the offending expression.

**Allocation step — Partial driver support**  
If driver data is missing for some target members, SAC now logs a warning instead of silently skipping those members. The warning appears in the execution log.

**Multi-Action — API step enhancements**  
- OAuth 2.0 authentication now supported directly in the API step configuration (previously required a custom header)
- Response body size limit increased from 1MB to 5MB

**Data Locking — Check Lock Status step**  
A new "Check Lock Status" step is now available in Multi-Actions. Use it to verify whether a data slice is locked before attempting to write, enabling conditional branching.

**Version Management — Selective copy**  
The "Copy Version" operation now supports filtering by dimension members — you can copy a subset of a version rather than the entire version.

**Parameters — Default value expressions**  
Parameter default values can now reference other parameters (e.g. set `EndPeriod` default = `StartPeriod` + 3 months). Previously defaults were static only.

---

## SAC 2025.16 — Data Actions and Planning Changes

**FOREACH — Nested iteration performance**  
Nested `FOREACH` loops now execute up to 40% faster for large dimension hierarchies due to query engine improvements.

**Copy Step — Offset time mapping**  
The Copy step now includes a built-in "period offset" configuration field, removing the need to use Advanced Formula for time-shifted copies (e.g. copy Jan Actual → Jan Budget with +12 month offset).

**Allocation — Allocation trace**  
A new allocation trace log is written to the execution log showing the driver values used for each target member. Helpful for debugging zero-allocation issues.

---

## SAC 2025.13 — Data Actions and Planning Changes

**Advanced Formula — RESULTLOOKUP cross-model performance**  
Cross-model `RESULTLOOKUP` calls are now batch-executed instead of row-by-row, significantly improving performance for formulas that reference driver models.

**Multi-Action — Step retry configuration**  
Individual steps can now be configured with an automatic retry count (1-3 retries) for transient failures, particularly useful for API steps.

---

## Deprecated / Removed Features

### Deprecated in 2025.19
- **Old Allocation UI (legacy)**: The original allocation configuration panel is deprecated. All allocations should be configured using the new Allocation step UI. SAP plans to remove the legacy UI in 2026.12.

### Removed
- _(None in current release)_

---

## Upcoming Changes (Preview)

> These are items SAP has announced as "planned" or "preview" — not yet generally available. Treat as indicative only until GA.

**Data Actions API — Bulk trigger**  
Ability to trigger multiple data actions in sequence via a single API call. ETA: 2026.Q1

**Advanced Formula — AI-assisted authoring**  
Generative AI suggestions for formula autocompletion directly in the SAC formula editor. Preview program open.

---

## Useful Links

- [SAP Help: What's New in SAC](https://help.sap.com/whats-new/cf0cb2cb149647329b5d02aa96303f56)
- [SAP Community: SAC Planning](https://community.sap.com/topics/cloud-analytics/planning)
- [SAP Road Map Explorer: SAC](https://roadmaps.sap.com/board?PRODUCT=73554900100800001024)
- [SAP Help: Data Actions](https://help.sap.com/docs/SAP_ANALYTICS_CLOUD/00f68c2e08b941f081002fd3691d86a7/af5e6fcec4514b20b0d97ea6e9b523f2.html)
- [SAP Help: Multi-Actions](https://help.sap.com/docs/SAP_ANALYTICS_CLOUD/00f68c2e08b941f081002fd3691d86a7/c5a63a0e7c1745a181baacc6b5fd8a53.html)
