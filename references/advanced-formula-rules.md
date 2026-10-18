# Advanced Formula — Calculation Rules and Scope

**Source**: SAP Help Portal — SAP Analytics Cloud Q3 2026 (2026.15)
**Source URL**: https://help.sap.com/docs/SAP_ANALYTICS_CLOUD/00f68c2e08b941f081002fd3691d86a7?locale=en-US&state=PRODUCTION&version=release
**Last updated**: 2026-10-18

---

## Four Things You Must Know Before Writing Any Formula

1. **Steps execute sequentially.** Data action steps run in order. Step B calculates based on the result of Step A, and Step C based on Step B. All intermediate results are held in buffer; data is committed to the version only after all steps succeed.

2. **All calculations work at leaf level only.** Advanced formulas perform calculations only on leaf-level members (members without children). Parent members and calculated members are excluded unless you use `CONFIG.READ_CALCULATED_MEMBER_VALUES = ON` (HANA Cloud tenants only).

3. **RESULTLOOKUP does not aggregate across unspecified dimensions.** If you don't define a dimension in RESULTLOOKUP, SAC returns one row per distinct value of that dimension — not a sum. This is different from FOX and other languages. Use `AGGREGATE_DIMENSIONS` explicitly if you want aggregation.

4. **Unspecified dimensions iterate automatically.** If your script doesn't set a scope for a dimension, SAC runs the calculation for every leaf member of that dimension. This is the root cause of unexpectedly large calculation scopes.

---

## Calculation Scope — The Most Critical Concept

The base scope of a step includes **all combinations of leaf members** across all dimensions. For a model with:
- 5 Measures × 200 Accounts × 150 SalesOffices × 48 Products × 120 Dates = **864 million combinations**

Writing `DATA() = 100` writes to all 864 million cells. You must narrow scope using MEMBERSET.

### Levels of Scope (narrowest overrides broadest)

```
1. MEMBERSET scope     — applies across the entire step
2. Condition scope     — applies inside IF or FOREACH blocks
3. Statement scope     — applies to a single DATA/RESULTLOOKUP line
```

### MEMBERSET Scope

```
// Only 7,200 cells: 1 measure × 1 account × 150 sales offices × 4 products × 12 months
MEMBERSET [d/Measures] = "Amount"
MEMBERSET [d/Account] = "Revenue"
MEMBERSET [d/Product] = ("PRD001", "PRD002", "PRD003", "PRD004")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
DATA() = RESULTLOOKUP([d/Version] = "Actual")
```

### Condition Scope (IF and FOREACH)

IF statements apply a dimension filter **within the block** only. FOREACH iterates over one member per calculation — each iteration has a single-member scope for that dimension.

```
MEMBERSET [d/Account] = ("Revenue", "Cost")
// Within this block, only Account = "Salaries" is in scope (overrides MEMBERSET)
IF [d/Account] = "Salaries" THEN
  DATA() = RESULTLOOKUP() * 1.05
ENDIF
```

---

## ELSEIF Behaviour (from 2021.02 onwards)

ELSEIF **excludes** the scope already covered by the preceding IF and ELSEIF blocks. This prevents double-writing.

```
IF [d/Date] = "202501" THEN
  DATA() = 100     // Scope: Date = 202501
ELSEIF [d/Region] = "EUR" THEN
  DATA() = 200     // Scope: Date ≠ 202501, Region = EUR
ELSE
  DATA() = 300     // Scope: Date ≠ 202501, Region ≠ EUR
ENDIF
```

### IF Restrictions

| Restriction | Example of what doesn't work |
|---|---|
| RESULTLOOKUP in IF needs a measure for models with Measures dimension | `IF RESULTLOOKUP([d/Product]="A") > 0 THEN` — missing measure |
| Measure filters can't use OR | `IF [d/Measures] != "Qty" OR [d/CC] = "Germany"` |
| Measure filter + non-measure filter → ELSEIF not allowed | See docs |
| `>`, `<` comparisons don't work with non-numeric values | `IF [d/Date] > "2025-03-15"` |
| `DAY() > DAY()` comparisons not supported | Use numeric comparisons instead |

---

## FOREACH — Iteration

FOREACH runs a calculation block once per leaf member of the specified dimension.

```
// Calculate bonus per employee
FOREACH [d/Employee]
  DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Sales") * 0.1
ENDFOR

// FOREACH.BOOKED — iterate only over members that have booked data
FOREACH.BOOKED [d/Product]
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
ENDFOR
```

**FOREACH vs DATA:** FOREACH is required when the calculated result of one iteration must feed into the next (e.g. carry-forward calculations). Use DATA directly when iterations are independent.

```
// Carry forward — FOREACH required because each month's Opening = previous month's Closing
FOREACH [d/Date]
  DATA([d/Movement] = "OPENING") = RESULTLOOKUP([d/Movement] = "CLOSING", [d/Date] = PREVIOUS())
ENDFOR
```

### FOR Loop (counter-based)

```
INTEGER @iCount
FOR @iCount = 1 TO 12
  // loop body
ENDFOR
```

### BREAK statement

```
// Stop when Revenue exceeds 200
FOREACH [d/Product]
  IF RESULTLOOKUP([d/Account] = "Revenue") > 200 THEN
    BREAK
  ENDIF
  DATA([d/Account] = "Processed") = 1
ENDFOR
```

---

## Variables

### FLOAT and INTEGER

```
// Variable definitions go at the top, after CONFIG settings
FLOAT @GrowthRate
INTEGER @Counter

// Usage
@GrowthRate = RESULTLOOKUP([d/Account] = "Rate") / 100
DATA([d/Account] = "Forecast") = RESULTLOOKUP([d/Account] = "Actual") * (1 + @GrowthRate)
```

### VARIABLEMEMBER — Virtual dimension member

Temporarily stores values from multiple dimension members in a single virtual slot.

```
// Define a virtual member
VARIABLEMEMBER #totalSales OF [d/Account]

// Calculate total
DATA([d/Account] = #totalSales, [d/Product] = "#") = RESULTLOOKUP([d/Account] = "Sales")

// Use in condition
IF RESULTLOOKUP([d/Account] = #totalSales, [d/Product] = "#") > 1000 THEN
  DATA([d/Account] = "Rebate") = RESULTLOOKUP([d/Account] = "Sales") * 0.1
ENDIF
```

---

## CONFIG Functions Reference

Global configuration functions apply to the **entire step**. Define them at the top of the script, before MEMBERSET.

### CONFIG.GENERATE_UNBOOKED_DATA

Controls whether empty (unbooked) source cells are included in copy operations.

| Setting | Behaviour |
|---|---|
| `OFF` (default) | Only booked (non-empty) source cells are copied. If source is empty and target has a value, target is unchanged. |
| `ON` | Empty source cells are treated as 0. If source is empty, target is set to 0. |

```
// Include zero-writes to ensure no stale data remains
CONFIG.GENERATE_UNBOOKED_DATA = ON
DATA([d/Date] = "202501") = RESULTLOOKUP([d/Date] = "202412")
```

### CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE

Controls sign handling for account type (INC/EXP/ASS/LEQ).

| Setting | Behaviour |
|---|---|
| `OFF` (default) | RESULTLOOKUP always returns the stored number, same as shown in a story table. |
| `ON` | SAC applies debit/credit sign based on account type. Revenue (INC) + Cost (EXP) → sign flip happens automatically. |

```
// Correct gross margin: Revenue (INC) + COGS (EXP)
// With ON: COGS is returned as negative, so addition gives correct margin
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
DATA([d/Account] = "GrossMargin") = RESULTLOOKUP([d/Account] = "NetRevenue")
                                  + RESULTLOOKUP([d/Account] = "CostOfGoodsSold")
```

**Without this flag:** NetRevenue (900) + COGS (700) = 1600 (wrong — both positive)
**With this flag:** NetRevenue (900) + COGS (−700) = 200 (correct gross margin)

This is one of the most common sources of incorrect formula results when mixing INC and EXP account types.

### CONFIG.TIME_HIERARCHY

Governs which time hierarchy the script uses.

```
CONFIG.TIME_HIERARCHY = CALENDARYEAR   // Use calendar year (Jan-Dec)
CONFIG.TIME_HIERARCHY = FISCALYEAR     // Use fiscal year
```

If not defined: SAC uses the model's default time hierarchy setting.

### CONFIG.HIERARCHY

Specifies which hierarchy to use for a dimension.

```
CONFIG.HIERARCHY [ModelName] = [d/Entity].[h/RegionalHierarchy], [d/Product].[h/ProductLines]
// Or inside a MODEL block:
MODEL [Finance]
  HIERARCHY = [d/Entity].[h/H1]
  ...
ENDMODEL
```

### CONFIG.TIME_ZONE_OFFSET

Adjusts the base time for `TODAY()` and other date functions.

```
CONFIG.TIME_ZONE_OFFSET = 5.5   // India (UTC+5:30)
CONFIG.TIME_ZONE_OFFSET = -8    // US Pacific (UTC-8)
```

### CONFIG.READ_CALCULATED_MEMBER_VALUES

Allows RESULTLOOKUP to read calculated members (account formulas, non-leaf members).

```
CONFIG.READ_CALCULATED_MEMBER_VALUES = ON
DATA([d/Entity] = "#") = RESULTLOOKUP([d/Entity] = "Europe")  // Europe is a calculated parent
```

**Requires**: SAC tenant on SAP HANA Cloud. Will error on non-HANA-Cloud tenants.

### CONFIG.MDXERRORLEVEL

Controls how MDX errors are handled.
- `0` = Silent (skip errors, continue)
- `1` = Warning
- `2` = Error (default in most cases)

---

## Member Selector Functions

### MEMBERSET

Sets the scope for a dimension across the entire step (or within its block).

```
MEMBERSET [d/Version] = {[d/Version].[Budget]}
MEMBERSET [d/Date] = "202501" TO "202512"    // Range
MEMBERSET [d/Product] = ("PRD001", "PRD002") // Explicit list
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")  // All leaf members under 2025
```

`MEMBERSET` can also be used inside a `MODEL` block to restrict scope when reading from a linked model.

### BASEMEMBER

Returns all **leaf-level** (base) members of a parent.

```
BASEMEMBER([d/Date].[h/YQM], "2025")
// Returns: {2025.01, 2025.02, ..., 2025.12}

BASEMEMBER([d/Date].[h/YQM], "2025.Q1")
// Returns: {2025.01, 2025.02, 2025.03}
```

**Critical use case**: Always use BASEMEMBER when referencing a time parent — writing to a parent member directly does nothing because data is stored at leaf level only.

### ELIMMEMBER

Excludes one or more members from a set.

```
// All leaf products except "Total"
ELIMMEMBER([d/Product].[LEVEL00].Members, [d/Product].[Total])

// Exclude multiple members
ELIMMEMBER([d/CostCenter].[LEVEL00].Members, [d/CostCenter].[Overhead], [d/CostCenter].[Shared])

// Exclude by attribute value
ELIMMEMBER([d/Entity], [d/Entity].[p/Consolidation], "Y")
```

### AGGREGATE_DIMENSIONS / AGGREGATE_WRITETO

Forces aggregation across a dimension before writing.

```
// Aggregate across Entity, write result to "#" (unassigned member)
AGGREGATE_DIMENSIONS [d/Entity]
AGGREGATE_WRITETO [d/Entity] = "#"
MEMBERSET [d/Product] = "Sparkling Water"
DATA([d/Date] = "202501") = RESULTLOOKUP([d/Date] = "202412") + 20
```

Without `AGGREGATE_DIMENSIONS`, the +20 would be added to each entity separately (e.g. Germany+20, France+20, UK+20 = 270 total). With it, all entity values are summed first (50+70+90=210), then +20 is added (=230), and written to the "#" member.

---

## Write-Back Functions

### DATA

The primary write-back function. Writes a calculated value to a specific cell.

```
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Price") * RESULTLOOKUP([d/Account] = "Qty")
DATA([d/Audit] = "LogicCalc") = RESULTLOOKUP([d/Audit] = "Input", [d/Date] = PREVIOUS(12))
DATA()  // Write to all cells in the current scope
```

### DELETE

Deletes (sets to blank/unbooked) values in the specified scope.

```
// Delete all values for a specific product in 2025
MEMBERSET [d/Product] = "Discontinued_SKU"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
DELETE()
```

### WRITETO

An alternative write syntax, used for specific cross-dimensional writes.

```
WRITETO([d/Entity] = "Consolidated", RESULTLOOKUP() * 1.1)
```

---

## RESULTLOOKUP — Reading Data

Reads data from the model. The most commonly used read function.

```
// Read from current scope (no overrides)
RESULTLOOKUP()

// Read from a specific dimension member
RESULTLOOKUP([d/Account] = "Revenue")

// Read from multiple specific members
RESULTLOOKUP([d/Account] = "Price", [d/Entity] = "*")

// Read from a different version
RESULTLOOKUP([d/Version] = "Actual")

// Read with time offset (1 month ago)
RESULTLOOKUP([d/Date] = PREVIOUS(1, "MONTH", [d/Date]))

// Read from previous year
RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"))
```

**Key rule**: RESULTLOOKUP does **not** aggregate unless you use `AGGREGATE_DIMENSIONS`. Each unspecified dimension returns separate values per member.

---

## LINK — Cross-Model Data Reading

Reads data from a **different model** (linked model).

```
// Basic cross-model read
DATA([d/Account] = "BudgetCost") = LINK([DriverModel], [d/Version] = "Actual", [d/Measures] = "Cost")

// Read from linked model with dimension mapping
MODEL [Sales]
  MEMBERSET [d/Product] = "SKU_001"
  MEMBERSET [d/Date] = "202501" TO "202512"
ENDMODEL
DATA([d/Account] = "Quantity") = LINK([Sales], [d/Version] = "Actual", [d/Measures] = "Quantity")
```

---

## MODEL Block — Cross-Model Scope

Defines the scope for a linked model. Required before using `LINK`.

```
MODEL [LinkedModelName]
  MEMBERSET [d/Date] = "202501" TO "202512"
  MEMBERSET [d/Product] = ("SKU_001", "SKU_002")
  HIERARCHY = [d/Entity].[h/Regional]
ENDMODEL

// Use the model in formula
IF LINK([LinkedModelName], [d/Version] = "Actual", [d/Measures] = "Stock") > 0 THEN
  DATA([d/Account] = "Available") = LINK([LinkedModelName], [d/Version] = "Actual", [d/Measures] = "Stock")
ENDIF
```

---

## Time Functions

| Function | Syntax | Description |
|---|---|---|
| `PREVIOUS` | `PREVIOUS(n, "MONTH", [d/Date])` | n periods before the current member |
| `NEXT` | `NEXT(n, "MONTH", [d/Date])` | n periods after the current member |
| `PREYEARLAST` | `PREYEARLAST([d/Date])` | Last period of the previous year |
| `PERIOD` | `PERIOD("20250304")` | Specific date member |
| `TODAY` | `TODAY()` | Current date (adjusted by CONFIG.TIME_ZONE_OFFSET) |
| `DATEDIFF` | `DATEDIFF([d/Date], %Param%, "QUARTER")` | Difference between two dates in given unit |
| `DAY` | `DAY([d/Date])` | Day number within month |
| `DAYSINMONTH` | `DAYSINMONTH([d/Date])` | Number of days in the month |

---

## Attribute Functions

```
// Read a dimension attribute value
ATTRIBUTE([d/Product].[p/Weight])

// Use attribute in condition
IF ATTRIBUTE([d/Product].[p/Factor]) > 2000 THEN
  DATA() = RESULTLOOKUP() * 1.5
ENDIF

// Filter by attribute value
IF [d/Account].[p/ACCTYPE] = ("AST", "LEQ") THEN
  // Only runs for Asset and Liability accounts
ENDIF
```

---

## SisterMember — Cross-Entity Copy

```
// Copy data from a related entity defined by an attribute
MEMBERSET [d/Entity].[p/Calculation] = "Y"
DATA([d/Entity] = [d/Entity].[p/SisterMember]) = RESULTLOOKUP()
```

---

## Performance Best Practices

### 1. Replace FOREACH with DATA where possible
`DATA` is batch-executed and faster. `FOREACH` is needed only when one iteration depends on the previous result (carry-forward).

```
// Slower
FOREACH [d/Product]
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// Faster — same result
DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
```

### 2. Avoid large ELSE scope
The ELSE clause inherits the full remaining scope. Large ELSE blocks are slow.

```
// Avoid
IF [d/Account] = "Price" THEN
  DATA() = 100
ELSE
  DATA() = 200   // This runs for ALL other accounts — potentially millions of cells
ENDIF

// Better: restrict the ELSE explicitly
IF [d/Account] = "Price" THEN
  DATA() = 100
ENDIF
MEMBERSET [d/Account] = "Cost"
DATA() = 200
```

### 3. Reduce RESULTLOOKUP calls
Each `RESULTLOOKUP` triggers a read from the data store. Cache into a variable member when the same value is read multiple times.

```
// Inefficient — reads Revenue twice
DATA([d/Account] = "Profit") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
DATA([d/Account] = "Margin%") = RESULTLOOKUP([d/Account] = "Revenue") / RESULTLOOKUP([d/Account] = "Sales")

// Better — store Revenue once
VARIABLEMEMBER #rev OF [d/Account]
DATA([d/Account] = #rev) = RESULTLOOKUP([d/Account] = "Revenue")
DATA([d/Account] = "Profit") = RESULTLOOKUP([d/Account] = #rev) - RESULTLOOKUP([d/Account] = "Cost")
DATA([d/Account] = "Margin%") = RESULTLOOKUP([d/Account] = #rev) / RESULTLOOKUP([d/Account] = "Sales")
```

### 4. Use MEMBERSET aggressively
Narrow scope as early as possible. Every unnecessary dimension combination multiplies execution time.

### 5. Prefer <> 0 over = NULL comparisons

SAC Advanced Formula has **no ISNULL(), NVL(), or NVL2() functions** — those are HANA SQL / FOX. Use direct NULL comparison or a zero guard instead.

```
// Valid — direct NULL comparison
IF RESULTLOOKUP([d/Account] = "Revenue") = NULL THEN
  DATA([d/Account] = "Margin") = 0
ELSE
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
ENDIF

// Faster when you just need a non-zero guard (note: treats NULL and 0 the same)
IF RESULTLOOKUP([d/Account] = "Revenue") <> 0 THEN
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue") - RESULTLOOKUP([d/Account] = "Cost")
ENDIF

// Safe division — guard against both NULL and zero
IF RESULTLOOKUP([d/Account] = "Headcount") <> 0 THEN
  DATA([d/Account] = "CostPerHead") = RESULTLOOKUP([d/Account] = "TotalCost") / RESULTLOOKUP([d/Account] = "Headcount")
ENDIF
```

Note: division by zero in SAC returns NULL silently — no runtime error.

### 6. Use FOREACH.BOOKED instead of FOREACH where applicable
`FOREACH.BOOKED` iterates only over members that have data, skipping empty cells.

---

## Common Script Patterns

### Carry-Forward (month-to-month)
```
MEMBERSET [d/Account] = "Headcount"
MEMBERSET [d/Movement] = ("CLOSING", "OPENING")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
FOREACH [d/Date]
  DATA([d/Movement] = "OPENING") = RESULTLOOKUP([d/Movement] = "CLOSING", [d/Date] = PREVIOUS())
ENDFOR
```

### Year-over-Year Growth
```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = "Revenue"
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"), [d/Version] = "Actual") * (1 + %GrowthRate%)
```

### Intercompany Elimination
```
CONFIG.GENERATE_UNBOOKED_DATA = OFF
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
MEMBERSET [d/Audit] = "10"
IF [d/Account] = ("BSA_IC_AR", "BSA_IC_AP") THEN
  DATA([d/Entity] = ELIMMEMBER([d/Entity].[h/Hierarchy], [d/Entity], [d/Interco_Entity], [d/Entity].[p/ELIMINATION], "Y"))
    = -RESULTLOOKUP()
ENDIF
```

---

## Data Action Tracing (Debugging Tool)

SAC's tracing tool lets you inspect intermediate results at specific **tracepoints** without stopping execution.

### How to use tracing
1. Open the data action script editor
2. Click in the **left margin** next to a line number to add a tracepoint (red filled circle = saved mode, empty circle = edit mode)
3. Click **Open Tracing Panel** in the toolbar
4. Run the data action — it executes fully; tracepoints don't pause execution
5. In the tracing panel, select a tracepoint to see the data state at that point and changes since the previous tracepoint

### Tracepoint rules
- Maximum 20 tracepoints per data action (including embedded DAs)
- Maximum 10 traced iterations within a FOREACH loop
- Tracepoints must be at the **beginning of a calculation** — SAC auto-moves them
- Tracepoints added inside FOREACH require setting the dimension member condition (the **Conditional Tracepoint** dialog)
- Tracepoints are not available in the Visual Tool — script editor only

### Tracing vs TRACE()
- **Tracing** (tracepoints): Visual tool in the SAC UI, shows a table view of data state at each point. Best for complex multi-step debugging.
- **TRACE()** function: Writes a value and label to the execution log (text output). Best for quick sanity-checks.

```
// TRACE() still works in all formula steps
TRACE(RESULTLOOKUP([d/Account] = "Revenue"), "Revenue before allocation")
```
