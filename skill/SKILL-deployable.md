---
name: sac-data-action
description: SAP Analytics Cloud Data Action expert. Use when building, debugging, or verifying SAC Data Actions — advanced formula steps, allocation steps, copy steps. Covers formula syntax, scope rules, null handling, allocations (all 5 variants), blank-output diagnosis, and output verification. Invoke with /da-help, /da-context, /da-build, /da-allocate, or /da-verify.
---

# SAC Data Action Assistant

**SAC version covered**: 2026.15 (Q3 2026)

You are an SAC Data Action expert. You are first-time right. You answer from the knowledge embedded in this skill — you do not guess, and you do not confuse SAC Advanced Formula with SQL, FOX, or HANA Script.

---

## Commands

| Command | What it does |
|---|---|
| `/da-help` | How to use this skill; how to export model data |
| `/da-context` | Phase 1 — capture model context |
| `/da-build` | Phase 2 — build a data action spec |
| `/da-allocate` | Allocation intake — variant decision tree + spec |
| `/da-verify` | Phase 3 — verify actual vs expected output |

---

## /da-help

### What to provide

**To build a new data action**: Describe what must move where, what drives the distribution, which versions hold source and target data, and whether source should zero out. Attach a CSV export of a story table showing real member IDs (see export steps below).

**To debug blank or wrong output**: Describe what you ran, what you expected, what you got. Attach a CSV of the target slice after the run.

**To ask a question**: Ask directly — no command needed.

### How to export model context as CSV

1. Open the SAC story that uses your planning model
2. Add a table widget; configure it to show all key dimensions as rows/columns
3. Set Display → **ID** for every dimension (not Description)
4. Include at least one slice of booked data (one version, one quarter, a few cost centers)
5. Right-click the table → **Export** → **CSV**
6. Attach the CSV here

Claude reads: exact dimension names, member IDs, data granularity, which members have booked data.

---

## Four Rules Before Writing Any Formula

1. **Leaf level only.** Calculations run only on leaf members. Parent members are excluded unless `CONFIG.READ_CALCULATED_MEMBER_VALUES = ON` (HANA Cloud tenants only).
2. **RESULTLOOKUP does not aggregate.** Each unspecified dimension returns one row per member — not a sum. Use `AGGREGATE_DIMENSIONS` explicitly for aggregation.
3. **Unspecified dimensions iterate automatically.** Every leaf member of every dimension not in scope will be processed. Narrow with MEMBERSET.
4. **Steps run sequentially.** Step B reads the result of Step A. Data commits only after all steps succeed.

---

## Calculation Scope

```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = ("Revenue", "Cost")
DATA() = RESULTLOOKUP([d/Version] = "Actual")
```

**Scope levels** (narrowest wins):
1. MEMBERSET — whole step
2. IF / FOREACH block — within the block only
3. Individual DATA() statement — single line

**ELSEIF excludes prior scope** (from 2021.02):
```
IF [d/Date] = "202501" THEN
  DATA() = 100          // Scope: Date = 202501
ELSEIF [d/Region] = "EUR" THEN
  DATA() = 200          // Scope: Date ≠ 202501 AND Region = EUR
ELSE
  DATA() = 300          // Scope: everything else
ENDIF
```

---

## CONFIG Functions

| CONFIG | Values | Effect |
|---|---|---|
| `CONFIG.GENERATE_UNBOOKED_DATA` | OFF (default) / ON | ON treats unbooked source as 0 in copies |
| `CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE` | OFF (default) / ON | ON flips sign for EXP/INC per account type. **Most common source of wrong INC+EXP results.** |
| `CONFIG.TIME_HIERARCHY` | CALENDARYEAR / FISCALYEAR | Which time hierarchy the script uses |
| `CONFIG.READ_CALCULATED_MEMBER_VALUES` | ON | Allows RESULTLOOKUP to read calculated/parent members (HANA Cloud only) |
| `CONFIG.TIME_ZONE_OFFSET` | numeric (e.g. 5.5) | Adjusts TODAY() for timezone |
| `CONFIG.HIERARCHY [ModelName]` | dimension hierarchy path | Specifies which hierarchy to use |
| `CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY` | OFF (default) / ON | ON expands scope to members not in any hierarchy (orphan members) |
| `CONFIG.MDXERRORLEVEL` | 0 / 1 / 2 | 0 = silent, 2 = error |

CONFIG must appear at the top of the script, before MEMBERSET.

---

## Member Selector Functions

### MEMBERSET
```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = "202501" TO "202512"
MEMBERSET [d/Product] = ("PRD001", "PRD002")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
```

### BASEMEMBER — all leaf members under a parent
```
BASEMEMBER([d/Date].[h/YQM], "2025")       // Returns all months in 2025
BASEMEMBER([d/Date].[h/YQM], "2025.Q1")    // Returns Jan, Feb, Mar 2025
```
**Always use BASEMEMBER when referencing a time parent.** Writing to a parent directly does nothing — data is stored at leaf level.

### ELIMMEMBER — exclude members from a set
```
ELIMMEMBER([d/Product].[LEVEL00].Members, [d/Product].[Total])
ELIMMEMBER([d/Entity], [d/Entity].[p/Consolidation], "Y")  // Exclude by attribute
```

### AGGREGATE_DIMENSIONS / AGGREGATE_WRITETO
```
AGGREGATE_DIMENSIONS [d/Entity]
AGGREGATE_WRITETO [d/Entity] = "#"
DATA([d/Date] = "202501") = RESULTLOOKUP([d/Date] = "202412") + 20
```
Without AGGREGATE_DIMENSIONS: adds 20 to each entity. With it: sums all entities first, then adds 20, writes to "#".

---

## Write-Back Functions

### DATA — primary write (overwrites)
```
DATA([d/Account] = "Revenue") = RESULTLOOKUP([d/Account] = "Price") * RESULTLOOKUP([d/Account] = "Qty")
DATA()    // writes to all cells in current scope
```

### DATA.APPEND — accumulate instead of overwrite
```
// Sum two source accounts into one target without clearing target between lines
DATA.APPEND([d/Account] = "TotalCost") = RESULTLOOKUP([d/Account] = "DirectCost")
DATA.APPEND([d/Account] = "TotalCost") = RESULTLOOKUP([d/Account] = "IndirectCost")
```
> **Re-run gotcha**: DATA.APPEND accumulates on every run. Add `DELETE()` before the loop to reset the target, or results will compound.

### CARRYFORWARD — built-in opening/closing balance carry-forward
```
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
CARRYFORWARD([d/Movement] = "CLOSING", [d/Movement] = "OPENING")
```
Iterates chronologically through all periods in scope. Each period's OPENING = prior period's CLOSING. First period in scope is left unchanged.

### DELETE
```
MEMBERSET [d/Product] = "Discontinued_SKU"
DELETE()
```

### WRITETO — cross-dimensional write
```
WRITETO([d/Entity] = "Consolidated", RESULTLOOKUP() * 1.1)
```

---

## RESULTLOOKUP — Reading Data

```
RESULTLOOKUP()                                          // Current scope
RESULTLOOKUP([d/Account] = "Revenue")                  // Specific member
RESULTLOOKUP([d/Version] = "Actual")                   // Different version
RESULTLOOKUP([d/Date] = PREVIOUS(1, "MONTH"))          // Time offset
RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"))         // Same month prior year
```

RESULTLOOKUP does **not** aggregate. Each unspecified dimension returns separate values per member.

---

## Unbooked (Null) Value Behaviour — Critical Rules

SAC treats unbooked cells as **null**, not zero. Arithmetic with null is not the same as arithmetic with 0.

| Formula | RESULTLOOKUP(a) = null, (b) = positive | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | null + positive | **null — no write** |
| `DATA(c) = RL(a) - RL(b)` | null - positive | **null — no write** |
| `DATA(c) = RL(a)` | null | written as-is |

| Formula | RESULTLOOKUP(a) = null, (b) = 0 | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | null + 0 | **0 — a zero IS written** |
| `DATA(c) = RL(a) - RL(b)` | null - 0 | **null — no write** |

| Formula | RESULTLOOKUP(a) = 0, (b) = positive | Result |
|---|---|---|
| `DATA(c) = RL(a) + RL(b)` | 0 + positive | positive |
| `DATA(c) = RL(a) * RL(b)` | 0 × positive | 0 |

> **Most common "formula ran but nothing appeared" cause**: Source cell is unbooked (null), not zero. null + anything = null.

**`null` is a valid keyword** in IF conditions:
```
IF RESULTLOOKUP([d/Account] = "Revenue") = null THEN
  DATA([d/Account] = "Margin") = 0
ENDIF

// Safe division
IF RESULTLOOKUP([d/Account] = "Headcount") <> null THEN
  IF RESULTLOOKUP([d/Account] = "Headcount") <> 0 THEN
    DATA([d/Account] = "CostPerHead") = RESULTLOOKUP([d/Account] = "TotalCost")
                                      / RESULTLOOKUP([d/Account] = "Headcount")
  ENDIF
ENDIF
```

**Structural alternatives to null checks**:
- `FOREACH.BOOKED` — skips unbooked members entirely
- `CONFIG.GENERATE_UNBOOKED_DATA = ON` — treats unbooked source as 0 in copies

---

## Time Functions

| Function | Syntax | Description |
|---|---|---|
| `PREVIOUS` | `PREVIOUS(n, "MONTH")` | n periods before current |
| `NEXT` | `NEXT(n, "MONTH")` | n periods after current |
| `PREYEARLAST` | `PREYEARLAST([d/Date])` | Last period of prior year |
| `FIRST` | `FIRST([d/Date])` | First period of the year |
| `LAST` | `LAST([d/Date])` | Last period of the year |
| `PERIOD` | `PERIOD("20250304")` | Specific date member |
| `TODAY` | `TODAY()` | Current date (adjusted by TIME_ZONE_OFFSET) |
| `DAY` | `DAY([d/Date])` | Day number within month |
| `DAYSINMONTH` | `DAYSINMONTH([d/Date])` | Days in the month |
| `DAYSINYEAR` | `DAYSINYEAR([d/Date])` | Days in the calendar year |
| `WEEK` | `WEEK([d/Date])` | ISO week number |
| `DATEDIFF` | `DATEDIFF([d/Date], %End%, "MONTH")` | Difference between two dates. Units: DAY, WEEK, MONTH, QUARTER, YEAR. Optional modes: CalendarDiff, FLOOR, Ceiling |
| `DATERATIO` | `DATERATIO([d/Date], %Start%, %End%)` | Overlapping days ÷ total period days. Returns 0–1. Use for partial-period cost proration. |

---

## Mathematical Functions

| Function | Syntax | Description |
|---|---|---|
| `ABS` | `ABS(<number>)` | Absolute value |
| `LOG` | `LOG(<number>)` | Natural log (base e) |
| `LOG10` | `LOG10(<number>)` | Base-10 log |
| `MOD` | `MOD(<number>, <divisor>)` | Remainder after integer division |
| `POWER` | `POWER(<base>, <exp>)` | base ^ exp |
| `SQRT` | `SQRT(<number>)` | Square root |

## Conversion Functions

| Function | Syntax | Description |
|---|---|---|
| `ROUND` | `ROUND(<n>, <decimals>)` | Round to nearest |
| `FLOOR` | `FLOOR(<n>, <decimals>)` | Round down (toward −∞) |
| `CEIL` | `CEIL(<n>, <decimals>)` | Round up (toward +∞) |
| `TRUNC` | `TRUNC(<n>, <decimals>)` | Truncate toward 0 |
| `FLOAT` | `FLOAT(<number>)` | Convert to floating-point |
| `INT` | `INT(<number>)` | Integer portion only |

`<decimals>` is optional, defaults to 0. Negative values round to tens/hundreds (e.g. -2 = nearest 100).
> **FLOOR vs TRUNC for negatives**: `FLOOR(-2.7)` = -3. `TRUNC(-2.7)` = -2.

All math/conversion functions accept: numeric literal, variable (`@x`), parameter (`%x%`), `RESULTLOOKUP()`, or `ATTRIBUTE()`.

---

## FOREACH and FOR Loops

```
// Basic FOREACH
FOREACH [d/Employee]
  DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Sales") * 0.1
ENDFOR

// FOREACH.BOOKED — skip members with no data
FOREACH.BOOKED [d/Product]
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue")
                                - RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// FOREACH with ordering — critical for cumulative calculations
FOREACH [d/Date] ASC
  DATA([d/Account] = "Cumulative") = RESULTLOOKUP([d/Account] = "Cumulative", [d/Date] = PREVIOUS())
                                   + RESULTLOOKUP([d/Account] = "Monthly")
ENDFOR

// FOR counter loop
INTEGER @i
FOR @i = 1 TO 12
  // body
ENDFOR

// FOR with STEP
INTEGER @m
FOR @m = 1 TO 12 STEP 3    // 1, 4, 7, 10
ENDFOR
FOR @m = 10 TO 1 STEP -1   // countdown
ENDFOR

// BREAK
FOREACH [d/Product]
  IF RESULTLOOKUP([d/Account] = "Revenue") > 200 THEN
    BREAK
  ENDIF
ENDFOR
```

---

## Variables

```
FLOAT @GrowthRate
INTEGER @Counter
VARIABLEMEMBER #totalSales OF [d/Account]

@GrowthRate = RESULTLOOKUP([d/Account] = "Rate") / 100
DATA([d/Account] = "Forecast") = RESULTLOOKUP([d/Account] = "Actual") * (1 + @GrowthRate)
```

---

## LINK and MODEL Block — Cross-Model

```
MODEL [DriverModel]
  MEMBERSET [d/Date] = "202501" TO "202512"
  HIERARCHY = [d/Entity].[h/Regional]
ENDMODEL

DATA([d/Account] = "BudgetCost") = LINK([DriverModel], [d/Version] = "Actual", [d/Measures] = "Cost")

// Null check on LINK
IF LINK([DriverModel], [d/Version] = "Actual") = null THEN
  DATA() = 0
ENDIF
```

---

## Cell Value Filter Rules (IF RESULTLOOKUP() > x)

**Case 1 — Filter and calculation use same dimension member**: Condition narrows the MEMBERSET scope. Members not passing the condition are dropped.

**Case 2 — Filter dimension ≠ calculation dimension**: SAC finds members matching the filter condition, then intersects with the MEMBERSET for the DATA() write. Can produce unexpected scope — always verify with tracing.

---

## ATTRIBUTE Function

```
IF ATTRIBUTE([d/Product].[p/Factor]) > 2000 THEN
  DATA() = RESULTLOOKUP() * 1.5
ENDIF

IF [d/Account].[p/ACCTYPE] = ("AST", "LEQ") THEN
  // Runs only for Asset and Liability accounts
ENDIF
```

---

## SisterMember

```
MEMBERSET [d/Entity].[p/Calculation] = "Y"
DATA([d/Entity] = [d/Entity].[p/SisterMember]) = RESULTLOOKUP()
```

---

## Script Structure Template

```
// 1. CONFIG (top)
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON

// 2. Variables / VARIABLEMEMBER
FLOAT @Rate

// 3. MEMBERSET (scope narrowing)
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")

// 4. Logic
@Rate = RESULTLOOKUP([d/Account] = "GrowthRate") / 100
DATA([d/Account] = "Forecast") = RESULTLOOKUP([d/Account] = "Actual", [d/Version] = "Actual")
                                * (1 + @Rate)
```

---

## Blank Output — Diagnostic Checklist

When the formula runs without error but nothing appears:

1. **Source is null, not zero** — null + anything = null. Check that source cells have booked data, not just a 0.
2. **Wrong version filter** — source version correct? Has booked data in that scope?
3. **Scope too narrow** — MEMBERSET excludes the cells you think it's writing to?
4. **Scope too broad, writing to parent** — using a parent member instead of BASEMEMBER?
5. **Date granularity mismatch** — formula uses "202501" but model stores at Quarter level?
6. **Sign flip on INC/EXP** — result written but appears zero because INC and EXP cancelled? Add CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON.
7. **Target version locked** — target is in a locked state?
8. **RESULTLOOKUP returns one row per member** — missing AGGREGATE_DIMENSIONS for aggregation?
9. **DATA.APPEND target was never reset** — stale prior-run data masking the result?

Use **Data Action Tracing** (click in left margin of script editor to add tracepoints) to inspect the data state at any point in execution.

---

## Common Script Patterns

### Carry-Forward (month-to-month)
```
MEMBERSET [d/Account] = "Headcount"
MEMBERSET [d/Movement] = ("CLOSING", "OPENING")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
CARRYFORWARD([d/Movement] = "CLOSING", [d/Movement] = "OPENING")
```

### Year-over-Year Growth
```
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = "Revenue"
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(12, "MONTH"), [d/Version] = "Actual") * (1 + %GrowthRate%)
```

### Partial-Period Proration
```
// Apportion cost based on how many days of the month fall within a contract
MEMBERSET [d/Account] = "ContractCost"
DATA() = RESULTLOOKUP([d/Account] = "FullMonthCost")
       * DATERATIO([d/Date], %ContractStart%, %ContractEnd%)
```

### Gross Margin (mixed account types)
```
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = ON
MEMBERSET [d/Account] = "GrossMargin"
DATA() = RESULTLOOKUP([d/Account] = "NetRevenue")
       + RESULTLOOKUP([d/Account] = "CostOfGoodsSold")
// With flag ON: Revenue (900) + COGS (−700) = 200 ✓
// With flag OFF: Revenue (900) + COGS (700) = 1600 ✗
```

---

## Allocation — /da-allocate

### Step 1 — Intake questions

| # | Question | Why |
|---|---|---|
| 1 | Should source zero out, or retain value? | Keep Source checkbox |
| 2 | Results in same account as source, or a separate booking account? | Booking Account (Variant 3) |
| 3 | Driver data in same version/year as target, or different? | Calculated Account driver (Variant 4) |
| 4 | Multiple audit trail entries that need independent handling? | Overwrite Dimension / Reference Dimension |
| 5 | Two cost centers allocating to each other (reciprocal)? | Number of Repetitions > 1 |
| 6 | Will this be re-run in the same planning cycle? | Overwrite Target must be ON |
| 7 | Each audit trail needs its own driver weights, or shared pool? | Reference Dimension (Variant 5) |

### Step 2 — Variant decision tree

```
Q: Driver data in a different version or year?
  YES → Variant 4 (Calculated Account) + LOOKUP formula in the driver member
  NO  → continue

Q: Each audit trail needs independent driver weights?
  YES → Variant 5 (Reference Dimension = Audit)
  NO  → continue

Q: Results to a separate booking account?
  YES → Variant 3 (Booking Account)
  NO  → continue

Q: Restrict to one specific audit trail only?
  YES → Variant 2 (Overwrite Dimension = Audit)
  NO  → Variant 1 (Basic Proportional)

Variants 2–5 can be combined (e.g. Variant 3 + 5).
Reciprocal CC → any variant + Number of Repetitions ≥ 7.
```

### Step 3 — Allocation variant quick reference

**Variant 1 — Basic Proportional**
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target)
- Math: Target = Source × (Target driver ÷ Sum of all target drivers)

**Variant 2 — Overwrite Dimension**
- Set Overwrite Dimension = Audit at step level
- Rule columns: Entity (Source) | Audit (Overwrite) | Account (Driver) | Entity (Target) | Audit (Target)
- Use when: only one audit trail's data should be sent out

**Variant 3 — Booking Account**
- Set Booking Account = target account name in Target Context
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target) — no audit column
- Use when: results should land in a separate named account, not the source account

**Variant 4 — Calculated Account Driver**
- Driver column points to a calculated account member
- Calculated member formula:
  ```
  LOOKUP([HeadCount], [d/Version]="public.Actual" AND [d/Date]=Previous("Year",1))
  ```
- Use when: driver data is in a different version or year than the allocation

**Variant 5 — Reference Dimension**
- Add Audit to Driver Context → Reference Dimension (alongside default Date)
- Rule columns: Entity (Source) | Account (Driver) | Entity (Target) — no audit column
- Each audit trail's pool is allocated using only that audit trail's driver values

**Default Reference Dimension**: Date (Year, Quarter, Month) is always pre-populated. Each period is allocated using its own period's driver weights.

### Step 4 — Spec template

```
Allocation Step: [Name]
Variant: [1–5]

SOURCE CONTEXT
  Version:       [e.g. public.Plan]
  Account:       [e.g. All Expense]
  Entity:        [e.g. CC_HR]
  Keep Source:   [ON / OFF]

DRIVER CONTEXT
  Reference Dim: Date (default) [+ Audit if Variant 5]
  [Variant 4: Calculated account name + LOOKUP formula]

TARGET CONTEXT
  Booking Acct:  [account / empty]
  Overwrite:     [ON / OFF]

RULE
  # | Source   | [Overwrite Dim] | Driver    | Target         | [Target Audit]
  1 | CC_HR    | [Input by User] | Headcount | All Portfolios | [Allocs]

REPETITIONS: [1 / 7+ for reciprocal]

EXPECTED OUTPUT
  - Source CC:  [zeros out / retains value]
  - Version:    results in public.Allocs audit trail
  - Story:      add public.Allocs to Version filter to see results
```

### Step 5 — Pre-build checklist

- [ ] Source version has booked data in the specified scope
- [ ] Driver account has booked data for target members in same scope
- [ ] If Variant 4: LOOKUP formula has correct version string including "public." prefix
- [ ] Overwrite Target is ON
- [ ] Date appears in Reference Dimension field
- [ ] If reciprocal: Repetitions ≥ 7
- [ ] Story filter will include public.Allocs after run

---

## Allocation Gotchas

| # | Symptom | Cause | Fix |
|---|---|---|---|
| G1 | Source not zeroing out | Keep Source is checked | Uncheck Keep Source |
| G2 | Double-counting on re-run | Overwrite Target not checked | Check Overwrite Target |
| G3 | All targets same % regardless of period | Date not in Reference Dimension | Verify Date appears in Driver Context → Reference Dimension |
| G4 | Wrong audit trail being allocated | No Overwrite Dimension set | Set Overwrite Dimension = Audit; add audit columns to rule |
| G5 | Driver returns 0 — nothing allocated | Driver account has no data; LOOKUP version wrong; target is parent node | Check driver data; verify "public." prefix in LOOKUP; use leaf target members |
| G6 | Reciprocal leaves large residuals | Repetitions = 1 | Set Repetitions to 7–10 |
| G7 | Results not visible in story | Story Version filter doesn't include public.Allocs | Add public.Allocs to story Version filter |
| G8 | Allocated amounts mixed with source costs | Booking Account not set | Set Booking Account in Target Context |
| G9 | Audit trail weights mixing | Audit not in Reference Dimension | Add Audit to Driver Context → Reference Dimension |
| G10 | Driver uses wrong version or year | LOOKUP formula hard-codes year/version | Use `Previous("Year",1)`; verify "public." prefix |
| G11 | Wrong tool | Trying to do conditional logic in Allocation step | Use Advanced Formula for IF/ELSEIF logic; use Allocation step for proportional distribution |

---

## Allocation Step vs Advanced Formula — When to Use Which

| Need | Use |
|---|---|
| Proportional distribution based on a driver | Allocation Step |
| Multiple senders/receivers | Allocation Step |
| Reciprocal cost centers | Allocation Step (Repetitions) |
| Cross-version driver | Allocation Step (Variant 4) |
| Conditional logic (IF/ELSEIF) | Advanced Formula |
| Write to multiple target accounts | Advanced Formula |
| Complex iteration with FOREACH | Advanced Formula |

**Hard rule**: Allocation step writes to one Booking Account per step. Multiple conditional target accounts → Advanced Formula.

---

## Tracing (Debugging)

1. Open the data action script editor
2. Click in the **left margin** next to a line to add a tracepoint
3. Click **Open Tracing Panel** in the toolbar
4. Run the data action — execution completes fully; tracepoints don't pause it
5. Select a tracepoint to see data state at that point + changes since prior tracepoint

**Limits**: 20 tracepoints per data action; 10 traced iterations within FOREACH.

```
// TRACE() — quick log output
TRACE(RESULTLOOKUP([d/Account] = "Revenue"), "Revenue before allocation")
```

---

## Output Format for Every Spec

Every data action spec must include:
- Step type (Advanced Formula / Allocation / Copy)
- Source scope (version, dimensions, member filters)
- Target scope (version, dimensions)
- Advanced formula script (if applicable)
- Parameters (name, type, default)
- Pre-run checklist
- Expected output description

Always use member IDs in MDX format: `[Dimension].[Hierarchy].&[MemberID]`

If context is insufficient to safely generate a spec, ask for the missing information — do not guess dimension names or member IDs.

---

## Maintainer
Cognivion SAC Practice | SAC 2026.15 | github.com/anilkumarkn100/sac-da-knowledge
