# Allocations — Reference

**Source**: SAP Help — SAP Analytics Cloud Allocations
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## What is an Allocation?

An Allocation step distributes a source value (the **key figure pool**) across a set of target dimension members using a defined method and optional **driver** (distribution basis).

Allocations are used when:
- Overhead costs need to be spread to cost centres proportionally
- Revenue targets need to be broken from region → country → product
- Budget amounts entered at a parent level need to be distributed to leaf level

---

## Allocation Architecture

```
Source (Pool)          Driver                    Target
─────────────────      ──────────────────        ──────────────────────
[Overhead Pool]        [Headcount by CC]    →    [Overhead per CC]
Single member or set   Measure with weights       Members receiving share
```

---

## Allocation Methods

### Even (Equal Split)
Divides the source value equally across all target members.
- No driver required
- Simple, but rarely reflects business reality

### Distribution Key (Proportional)
Allocates based on a **driver measure** (also called the distribution key or basis).

Formula:  
`Target Member Share = (Driver Value for Target) / (Total Driver Value) × Source Pool`

**Requirements:**
- Driver measure must have values for **every target member**
- Driver values must be at the **same granularity** as the target
- If any driver value is zero or missing, that member receives zero allocation

**Most common cause of blank allocation output:**
Driver data missing at the required granularity level.

### Formula
Custom allocation logic written as an Advanced Formula expression. Gives full control but requires scripting.

---

## Allocation Step Configuration (UI)

1. Add Allocation step to data action
2. Configure:
   - **Source**: Measure + dimension members that form the pool
   - **Target**: Measure + dimension members to receive allocation
   - **Method**: Even / Distribution Key / Formula
   - **Driver** (if Distribution Key): Select the driver measure
   - **Time filter**: Match to the period being allocated
   - **Version**: Specify source and target versions explicitly

---

## Allocation in Advanced Formula

You can replicate allocation logic manually for more control:

```
// Proportional allocation of overhead to cost centres
FOREACH(
  [d/CostCenter].[LEVEL00].Members,
  {
    [d/Measures].[AllocatedOverhead] =
      [d/Measures].[OverheadPool] *
      (
        [d/Measures].[Headcount] /
        RESULTLOOKUP(
          [d/CostCenter].[TOTAL],
          [d/Measures].[Headcount]
        )
      );
  }
)
```

**Note**: Division by zero risk — always check that the total driver value is non-zero before allocating.

---

## Granularity Considerations

This is the **most common cause of blank allocation results**.

| Scenario | Result |
|---|---|
| Source at Year, driver at Month, target at Month | ✅ Works if time dimension properly mapped |
| Source at Year, driver at Year, target at Month | ⚠️ Target months get zero — driver not at same granularity |
| Source at Month, driver at Year | ⚠️ Driver totals don't match source periods — allocation may be wrong |
| Driver missing for some target members | Those members receive zero |

**Rule**: Driver data must exist at **exactly** the granularity of the target dimension intersections.

---

## Multi-Level Allocation

When distributing across multiple levels (e.g. Region → Country → Product), run allocations in sequence:

1. Step 1: Region → Country (using country-level driver)
2. Step 2: Country → Product (using product-level driver)

Each step reads the result of the previous step as its source.

Alternatively, use a single formula-based allocation with nested `FOREACH`.

---

## Driver Data Management

Driver data is typically imported from HR systems or maintained manually in a separate driver model or version.

Best practice:
- Keep driver data in the **same model** as the planning data (simplifies scope)
- Or use `MODEL()` function to reference a separate driver model
- Update driver data before running allocations
- Version-tag driver data (e.g. use `Actual` headcount for budget allocation)

---

## Common Allocation Gotchas

### Gotcha 1: Zero driver values → zero allocation
If any target member's driver measure is 0 or blank, it receives no allocation. The total may not balance to the source.

**Diagnostic**: Check driver data completeness before running allocation.

### Gotcha 2: Overlapping source and target members
If a member appears in both the allocation source and target set, circular allocation occurs.

**Fix**: Use `ELIMMEMBER` to exclude the source member from the target set.
```
ELIMMEMBER([d/CostCenter].[LEVEL00].Members, [d/CostCenter].[OverheadPool])
```

### Gotcha 3: Time granularity mismatch
Source pool at quarter, driver at month, target at month → allocation gets distributed using total driver (year or all periods) instead of period-specific driver.

**Fix**: Always match driver granularity to target granularity. If needed, run one allocation step per period.

### Gotcha 4: Version not in context
Allocation writes to a version that's not in the data action's filter → silent blank output.

**Fix**: Explicitly add source and target versions to the data action filter.

### Gotcha 5: Driver totals to zero
When all driver values are zero, division by zero means no allocation occurs.

**Fix**: Add a guard condition in formula-based allocation:
```
IF(
  RESULTLOOKUP([d/CostCenter].[TOTAL], [d/Measures].[Headcount]) <> 0,
  // allocation formula,
  0
)
```

---

## Allocation vs. Copy Step

| | Allocation | Copy |
|---|---|---|
| Distributes a total | ✅ Yes | ❌ No (copies as-is) |
| Requires driver data | Distribution Key only | No |
| Multi-to-one | ✅ Yes (many targets, one source) | No |
| One-to-many | ✅ Yes | Possible with FOREACH |
| Transformation | Yes (via formula) | Copy factor only |
| Complexity | Higher | Lower |
