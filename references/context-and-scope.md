# Context and Scope — Reference

**Source**: SAP Help — SAP Analytics Cloud Data Actions Context
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## Why Context is the #1 Cause of Blank Output

The single most common reason a data action produces no visible result is **incorrect or missing model context**. The data action runs without error, but writes nothing because the scope it tries to write to does not exist in the model's data, or the dimensions do not align with how the model actually stores data.

This reference explains exactly how SAC determines what data to read and write, and how to avoid context failures.

---

## What is "Context"?

In SAC planning, **context** is the complete set of dimension member values that identify a single data cell. Think of it as the address of a cell in a multi-dimensional cube.

For a data action to write data, it must:
1. Have a valid context for every dimension in the model
2. That context must be at the correct **granularity** (leaf level vs. aggregated)
3. The context must exist as a valid data cell in the model's structure

If any dimension is missing from the context, SAC either:
- Uses a default (which may be wrong), or
- Silently skips the write

---

## Context Sources

Context flows into a data action from multiple sources, applied in this priority order:

```
1. Explicit MEMBERSET in the formula (highest priority)
2. Data action's own filter configuration
3. Story filter (when triggered from a story)
4. Model defaults / ALL members
```

The final effective context for each dimension is the **intersection** of all these sources.

### If the intersection is empty → no data written
Example: Story filter says Version = Budget, data action filter says Version = Actual, formula says MEMBERSET(Version, Forecast) → these don't intersect → no writes.

---

## Dimensions Every Model Has

All SAC planning models include these built-in dimensions:

| Dimension | Technical ID | Notes |
|---|---|---|
| **Account** (Measures) | `[d/Measures]` | Mandatory; holds all measures/KPIs |
| **Date/Time** | `[d/Date]` or `[d/Time]` | Mandatory; calendar hierarchy |
| **Version** | `[d/Version]` | Mandatory; planning versions |
| **Audit** | `[d/Audit]` | System dimension for booked data; usually `[d/Audit].[ACTUAL]` |
| **Currency** | `[d/Currency]` | If model uses currency |

Plus all custom dimensions defined at model build time (Entity, CostCentre, Product, etc.).

---

## The Audit Dimension

The **Audit dimension** is a system dimension that distinguishes **booked data** from other record types.

Most planning data lives in:  
`[d/Audit].[ACTUAL]` — the default booked data category

Other audit values are used for:
- Opening balances
- Carry-forward records
- Manually adjusted records

**Key rule**: If your data action does not include `[d/Audit].[ACTUAL]` in its context, it will not read or write any booked data.

This is commonly set implicitly through the model's default, but if you see all-blank results and everything else looks correct, **explicitly add `[d/Audit].[ACTUAL]` to your MEMBERSET**.

---

## Time Dimension Granularity

The time dimension is the most common source of context failures.

### Granularity levels
- **Year** — `2025`
- **Quarter** — `2025.Q1`
- **Month** — `2025.01`  (or `Jan 2025`)
- **Week** — `2025.W01`
- **Day** — `2025.01.15`

**The model's data is stored at a specific granularity** — determined at model creation. Data actions must read and write at **the same granularity** as the stored data.

### The classic mismatch
```
Model stores data at: Month
Formula references: [d/Time].[2025]   (year-level member)
Result: no data read, no data written
```

The year-level member `2025` is an **aggregation** of month members — it does not have its own data cell. Writing to it does nothing.

**Fix**: Use `BASEMEMBERS([d/Time].[2025])` to get the actual leaf-level members (months).

### How to find the model's time granularity
1. Open the model in the SAC model editor
2. Check the Date dimension configuration — the "Base Period" setting shows the stored granularity
3. Or export a small slice of data and examine the time member format

---

## Account / Measures Dimension

The Account (Measures) dimension holds all KPIs. There are two account types:

| Type | Description | Sign convention |
|---|---|---|
| **INC (Income)** | Revenue, income lines | Positive = credit |
| **EXP (Expense)** | Cost, expense lines | Positive = debit |
| **ASS (Asset)** | Asset accounts | Standard balance |
| **LEQ (Liability/Equity)** | Liability and equity | Standard balance |

### Sign convention gotcha
When copying from an INC account to an EXP account (or vice versa), SAC **inverts the sign** automatically because the account types differ.

This is one of the most confusing blank/wrong output causes:
- Copy a revenue figure (INC) to a cost driver (EXP) → signs invert
- The result appears negative when positive was expected

**Fix**: Be aware of account types in source and target. If sign inversion is not desired, multiply by `-1` in the formula.

---

## Model Context Sheet

The **Model Context Sheet** (see `templates/model-context-sheet.md`) is the tool for capturing a model's context before building data actions.

Required fields to capture:
1. All custom dimensions and their member structures
2. The time dimension granularity (what level is data stored at)
3. Which version members exist and their types
4. Which account members exist and their types (INC/EXP/ASS/LEQ)
5. Whether the model uses Audit dimension and what values
6. Whether the model uses Currency dimension

Fill the Model Context Sheet for a new model before writing any data actions.

---

## Scope Restriction Functions

Use these in Advanced Formula to explicitly set context for a dimension:

### MEMBERSET
```
MEMBERSET([d/Version], {[d/Version].[Budget]});
{
  // all code here runs in Budget version context
}
```

### BASEMEMBER
```
// Get all leaf-level time members under a parent
BASEMEMBER([d/Time].[2025.Q1])
// Returns: {2025.01, 2025.02, 2025.03}
```

### ELIMMEMBER
```
// Exclude a specific member from a set
ELIMMEMBER([d/CostCenter].[LEVEL00].Members, [d/CostCenter].[CC_TOTAL])
```

---

## Context Failure Diagnostic Steps

When a data action produces blank output:

### Step 1: Check version context
- Is the target version in the data action's filter?
- Is the target version locked (see `references/data-locking.md`)?

### Step 2: Check time granularity
- What granularity does the model store data at? (Check model editor)
- Are formula references at the same level (e.g. both at month)?
- Replace absolute time references with `BASEMEMBERS()` to ensure leaf-level targeting

### Step 3: Check dimension scope
- Does the formula reference all required custom dimensions?
- Are there dimensions in the model not mentioned in the formula? SAC uses the context filter — if no filter exists for that dimension, it may use ALL members or default

### Step 4: Check Audit dimension
- Add explicit `MEMBERSET([d/Audit], {[d/Audit].[ACTUAL]})` to rule out audit scope issues

### Step 5: Check Account types
- Are source and target accounts the same type (INC/EXP)?
- If different, is sign inversion expected?

### Step 6: Use TRACE
```
TRACE([d/Measures].[Revenue], "Revenue value at this context");
```
Run the data action and check the execution log for TRACE output. If the log shows the right value but data doesn't appear in the story, the write is being blocked (locked, wrong version, wrong member).

### Step 7: Narrow the scope
Restrict to a single member across all dimensions and verify a write occurs. Then progressively broaden the scope.

---

## Key Vocabulary

| Term | Meaning |
|---|---|
| **Context** | The complete set of dimension members identifying a data cell |
| **Granularity** | The lowest level at which data is stored in a dimension |
| **Leaf member** | A dimension member with no children (base level) |
| **Parent member** | A dimension member that aggregates children; data is not stored at parent level |
| **Booked data** | Actual stored data; usually `[d/Audit].[ACTUAL]` |
| **Scope** | The subset of a dimension's members included in an operation |
| **Silent failure** | A write that produces no error but also no data — caused by context mismatch |
