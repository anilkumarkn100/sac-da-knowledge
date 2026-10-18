# Data Action Step Types — Reference

**Source**: SAP Help — SAP Analytics Cloud Data Actions
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## Step Types Overview

| Step Type | Use Case | Notes |
|---|---|---|
| Copy | Move/copy data between dimensions, versions, time periods | Most common first step |
| Advanced Formula | Complex calculations using MDX-like syntax | Use `TRACE()` for debug |
| Allocation | Rule-based distribution of values using drivers | Needs driver data in model |
| Currency Conversion | FX conversion using stored rate table | Requires rate table model |
| Embedded Data Action | Call another data action as a step | Reuse common logic |
| API Step | Call external HTTP API | Multi-action only |
| Version Management | Publish, copy, delete versions | Multi-action only |
| Data Locking | Lock or unlock data slices | Multi-action only |

---

## Copy Step

A Copy step reads data from a source slice and writes it to a target slice. It is the most common first step in planning data actions.

### Key configuration fields

| Field | Description |
|---|---|
| **Source Version** | The planning version to read from (e.g. `Actual`, `Budget`) |
| **Target Version** | The planning version to write to |
| **Source Members** | Explicit member list or `BASEMEMBERS` of a parent |
| **Target Members** | Where the copied data lands; often same dimension, different member |
| **Copy Factor** | Optional multiplier applied to copied values (e.g. 1.0 = 100%) |
| **Aggregation** | If source is at higher granularity, define how to split |
| **Clear before copy** | Whether to zero out the target slice before writing |

### Common copy patterns

**Copy Actual to Budget seed:**
- Source Version: `Actual`, Time: `BASEMEMBERS([d/Time].[2024.Q4])`
- Target Version: `Budget`, Time: offset to next year equivalent period
- Copy Factor: 1.0 or growth rate parameter

**Adjust a budget figure:**
- Source Version: `Budget`
- Target Version: `Budget` (same version, different member)
- Apply formula via Advanced Formula step instead — Copy does not transform values

### Gotchas
- If Source and Target dimensions don't align in granularity, copy result is blank. Match granularity exactly or use `BASEMEMBERS`.
- Copy does **not** cascade: copying a parent member does not copy its children. Use `BASEMEMBERS` explicitly.
- `Clear before copy` with too-broad a scope wipes more data than expected. Scope carefully.
- Version filter must include both source and target version — if only one version is in context, the other slice returns nothing.

---

## Advanced Formula Step

An Advanced Formula step uses a scripting language to read and write planning data. It is the most flexible step type.

### Script structure

```
// Every advanced formula step runs in a model context.
// The MODELCONTEXT block is optional but recommended for clarity.

FOREACH(
  [d/CostCenter].[LEVEL00].Members,   // Iteration dimension
  {
    [d/Measures].[CostBudget] = 
      [d/Measures].[CostActual] * (1 + PARAM.GrowthRate);
  }
)
```

### Execution model
- The step executes once per combination of **context dimensions** not explicitly scoped in the formula.
- Data is read from and written to the **planning buffer** — changes are only committed when the data action completes successfully.
- Failed steps roll back all changes in that data action run.

### Common functions

| Function | Purpose |
|---|---|
| `FOREACH(set, {block})` | Iterate over a set, executing the block for each member |
| `MEMBERSET(dim, set)` | Restrict the scope of a dimension within the block |
| `BASEMEMBER(member)` | Get the leaf-level descendant set of a parent |
| `ELIMMEMBER(set, member)` | Exclude one or more members from a set |
| `AGGREGATE_DIMENSIONS(dims)` | Aggregate result across specified dimensions |
| `WRITETO(target, value)` | Write a value to a specific cell |
| `MODEL(model_name)` | Reference another model in the formula |
| `RESULTLOOKUP(member_refs)` | Read data from the model |
| `LINK(target, source)` | Copy cell values between coordinates |
| `TRACE(value, label)` | Write debug output to the execution log |
| `CONFIG.MDXERRORLEVEL` | Control how MDX errors are handled (`0` = silent) |
| `CONFIG.COPYMAPPING` | Define dimension mapping for cross-model copies |

### Parameters in Advanced Formula

Parameters are defined on the data action and passed in at runtime.

```
// Reading a parameter in the formula
PARAM.VersionFrom          // Member-type parameter
PARAM.GrowthRate           // Number-type parameter  
PARAM.EffectiveDate        // Datetime-type parameter
PARAM.Scenario             // String-type parameter
```

### Debug with TRACE

```
TRACE([d/Measures].[Revenue], "Revenue for member X");
```

Output appears in the data action execution log (accessible after the run).

---

## Allocation Step

An Allocation step distributes a total (the **key figure**) across a set of target members using a **driver** (also called a basis).

### Key configuration fields

| Field | Description |
|---|---|
| **Key figure** | The measure/account to be allocated |
| **Source members** | Where the total comes from (the "pool") |
| **Target members** | The members to receive the allocation |
| **Allocation method** | How to distribute: Even, Distribution Key, or Formula |
| **Distribution key** | Another measure that defines proportional weights |
| **Target version** | Where to write the allocated values |

### Allocation methods

**Even** — Divides the total equally across all target members.

**Distribution Key** — Allocates proportionally based on another measure's values.
- If driver data is zero for all targets, result is blank/zero (no allocation occurs)
- Ensure driver data exists at the same granularity as target

**Formula** — Allocates using a custom Advanced Formula expression.

### Gotchas
- Driver data must exist at the same time granularity as the target. Month-level allocation with year-level drivers = blank.
- If allocation target and source overlap (same member appears in both), circular allocation occurs — SAC will error or produce unexpected results.
- Allocation results are always stored at leaf level. Parent-level allocation targets are automatically expanded to base members.
- Zero or missing driver values = zero allocated weight = no data written. This is the most common cause of blank allocation results.

---

## Currency Conversion Step

Converts values from a source currency to a target currency using a rate table stored in a separate model.

### Requirements
- A **rate model** must exist in the SAC tenant with Source Currency, Target Currency, Rate Type, and Period dimensions
- The planning model must have a currency dimension

### Configuration fields

| Field | Description |
|---|---|
| **Rate Model** | The model storing FX rates |
| **Rate Type** | e.g. Closing, Average, Budget rate |
| **Source Currency Dimension** | Dimension in planning model holding source currency |
| **Target Currency** | The currency to convert to |
| **Time mapping** | How planning periods map to rate periods |

---

## Embedded Data Action Step

Executes another data action as a sub-step. Useful for reusing common logic.

### Notes
- Parameters of the called data action can be passed from the parent
- The embedded data action runs in the same transaction — if it fails, the parent rolls back
- Circular references (A calls B which calls A) are not supported and will error at save time

---

## Parameters

Parameters allow dynamic configuration at runtime. Define them on the data action; pass values when triggering from a story or API.

### Parameter types

| Type | Use | Example value |
|---|---|---|
| **Member** | Single dimension member | `[d/Version].[Budget]` |
| **Number** | Numeric constant | `1.05` |
| **String** | Text value | `"Budget_2025"` |
| **Datetime** | Date/time value | `2025-01-01` |

### Defining parameters (UI)
1. Open data action → Parameters tab
2. Add parameter: Name, Type, Default value, Required flag
3. Reference in formula as `PARAM.ParameterName`

### Passing parameters from a story
In the story's data action trigger dialog, map story filter values or input controls to each parameter.

---

## Execution Flow

1. Data action is triggered (manually, from story, or via API)
2. SAC validates parameter values
3. Steps execute sequentially in defined order
4. Each step reads from and writes to the **planning buffer**
5. If all steps succeed → changes committed to the model
6. If any step fails → entire data action rolled back (no partial writes)

### Execution log
After each run, an execution log is available showing:
- Step execution times
- Row counts processed
- TRACE() output
- Error messages

Access: Data Action → Run History → select run → View Log
