# SAC Allocation Step — Complete Reference

**Sources**:
- SAP Community Blog Series by jraymond15 (Parts 1–5): Basic, Overwrite Dimension, Booking Account, Calculated Account, Reference Dimension
- SAP Learning: "Creating Data Actions for Allocations" + "Performing Allocations"
- SAP Help Portal Q3 2026 (2026.15)

**Last updated**: 2026-10-18

---

## What Is an Allocation?

An allocation distributes costs from a **sender** (source entity/cost center) to **receivers** (target entities/cost centers) proportionally based on a **driver** (e.g. headcount, kilowatt hours, number of people).

**Use cases:**
- Distribute overhead cost centers (HR, IT, Utilities) to operating units
- Allocate costs based on headcount, floor space, kilowatt hours
- Spread annual budget to periods based on historical trends
- Handle reciprocal service allocations between cost centers

An **Allocation Process** = one or more **Allocation Steps** (structure) + one or more **Allocation Rules** per step (specific sender/receiver pairs).

---

## The Allocation Step — Four Sections

When you add an Allocation step to a data action, the configuration panel has four collapsible sections:

### 1. Source Context

| Field | Description |
|---|---|
| **Data Source** | Planning model — inherited from the data action, not configurable per step |
| **Filters** | Chip-based filters restricting which dimension members are in scope. Add chips for Version, Measures, Date, Account, etc. Version is mandatory (set it here or via parameter) |
| **Write Mode — Keep Source** | Checkbox. When checked, source values are **copied** to targets and the source cell retains its value. When unchecked (default), source value is **moved** — source becomes 0 after allocation |

> **Keep Source tip**: Use Keep Source = off when the source cost center should be zeroed out (pass-through). Use Keep Source = on when you want the source to retain its value (e.g. allocating from a reference version without changing it).

### 2. Driver Context

| Field | Description |
|---|---|
| **Reference Dimension** | Dimension(s) whose members are treated as independent allocation pools. **Date (Year, Quarter, Month) is pre-populated by default.** This means each time period is allocated separately using its own driver weights. Add further reference dimensions (e.g. Audit) when each member of that dimension should be weighted independently |

> **Why Reference Dimension matters**: Without it, driver values for all members of a dimension are summed first, then each member gets that combined weight. With it, each member uses only its own driver data. See the worked example in the Reference Dimension section below.

### 3. Target Context

| Field | Description |
|---|---|
| **Booking Account** | Optional. Write allocation results to this account instead of the source account. Leaves source account values unchanged. Use when you need to track allocation amounts separately (e.g. "HR Allocation In" vs "Labor") |
| **Write Mode — Overwrite Target** | Checkbox. When checked, existing target values are **deleted** before writing new allocation results. Use when re-running an allocation to avoid double-counting |

### 4. Allocation Rules

The rules grid defines which specific source → target pairs participate, and which account provides the driver weight.

**Number of Repetitions**: Controls iterative allocation for reciprocal scenarios.
- Default = 1 (single pass)
- Maximum = 10
- Use > 1 when two cost centers allocate to each other (reciprocal). The process repeats until residuals approach zero or the repetition limit is reached.

**Rule columns** (vary by variant — see each variant below):

| Column | Meaning |
|---|---|
| Source Dimension | The sending entity/cost center. Cannot be Version |
| Driver Dimension | The account whose values determine proportional weights |
| Target Dimension | The receiving entity/cost center. Cannot be Version or Account |

> You can select more than one source dimension and more than one target dimension per rule row. You can also use parameters instead of hard-coded members.

---

## Allocation Variants

### Variant 1 — Basic Proportional Allocation

The standard case. Source sends to targets proportionally based on a driver account.

**When to use**: Distribute a cost pool to multiple receivers using a simple numeric driver.

**Allocation Step configuration:**
- Source Context: Filters set for Version, Measures, Accounts to allocate
- Keep Source: unchecked (source zeroes out)
- Reference Dimension: Date (default — each month allocated separately)
- Booking Account: empty (results post to same account as source)
- Overwrite Target: checked (clean re-run)

**Allocation Rule columns:**
```
Entity (Source) | Account (Driver) | Entity (Target)
Cost Collector  | Headcount        | All Portfolios
```

**Math:**
```
Target amount = Source amount × (Target driver value / Sum of all target driver values)

Example: Source = $62,500, Driver totals = 100
  P1 has HC = 10 → receives $62,500 × 10/100 = $6,250
  P2 has HC = 30 → receives $62,500 × 30/100 = $18,750
  P3 has HC = 20 → receives $62,500 × 20/100 = $12,500
  P4 has HC = 40 → receives $62,500 × 40/100 = $25,000
```

**Results**: Written to version **public.Allocs** (the Allocs audit trail) by default.

---

### Variant 2 — Overwrite Dimension

Restricts the allocation to source records that match a specific audit trail member, and writes results tagged to a specific target audit trail.

**When to use**: Your source data has multiple audit trail entries (e.g. "Input by User" and "System Generated") and you only want to allocate from one of them.

**Allocation Step configuration:**
- Source Context: same as Basic
- Keep Source: typically unchecked
- **Overwrite Dimension**: set to **Audit** dimension — this restricts which source audit records are sent out

**Allocation Rule columns:**
```
Entity (Source) | Audit (Overwrite) | Account (Driver) | Entity (Target) | Audit (Target)
Cost Collector  | Input by User     | Headcount        | All Portfolios  | Allocs
```

The Audit (Overwrite) column restricts which audit trail's source data is used. The Audit (Target) column controls which audit trail receives the result.

> **Gotcha**: The driver data is read from the **target** members' records, not the source. The allocation reads headcount from the target entities to determine proportion.

---

### Variant 3 — Booking Account

Posts allocation results to a **named account** instead of the source account. Source account values remain unchanged; a separate "allocation in" account holds the distributed amounts.

**When to use**: You need to track allocation amounts separately from source costs. Common for HR cost allocations where you want to preserve original HR expenses while also showing how they were distributed.

**Allocation Step configuration:**
- Source Context: Filter by the accounts you want to allocate (e.g. "All Expense")
- Keep Source: unchecked (source cost center zeroes out)
- **Booking Account**: set to the receiving account (e.g. "Allocation (In) [Exp]" or "Human Resources")
- Overwrite Target: checked

**Allocation Rule columns:**
```
Entity (Source) | Account (Driver) | Entity (Target)
Cost Collector  | Headcount        | All Portfolios
```

Note: Booking Account is NOT a column in the rule — it is set at the step level in Target Context.

**Result**: The source account values are sent out; they land in the Booking Account on the target entities. The source CC shows a negative offset in the booking account row (showing how much was sent out), and the target CCs show positive values in the booking account row.

**Key benefit**: Targets can see both their original costs AND the allocated-in amounts separately. A comparison between actual and plan remains valid because the original cost categories are preserved.

---

### Variant 4 — Calculated Account as Driver

Uses a **calculated account member** (a formula-based member) as the driver instead of a stored data account. This enables cross-version or cross-period driver lookups.

**When to use**: Driver data lives in a different version or year than the allocation target. Common for allocating budget based on actual headcount from prior year.

**Calculated account member formula examples:**

```
// Hard-coded prior year — headcount from Actual version, 2023
Account PriorYearActualHeadCount =
  LOOKUP([Number of People],[d/Version]="public.Actual" AND[d/Date]="2023")

// Dynamic prior year — uses Previous() function for year offset
Account PriorYearPCs =
  LOOKUP([Number of PCs],[d/Date]=Previous("Year",1)AND[d/Version]="public.Actual")

// Cross-version lookup (from blog example)
Account CalcHC =
  LOOKUP([HC], [d/Version]="public.Plan V1")
```

**Allocation Step configuration:**
- Source Context: standard
- Reference Dimension: Date (default)

**Allocation Rule columns:**
```
Entity (Source) | Account (Driver)          | Entity (Target)
Cost Collector  | CalcHC (calculated member)| All Portfolios
```

The driver column points to the calculated account member — SAC evaluates the LOOKUP formula at allocation time to get the driver values.

> **Tip**: Use `Previous("Year", 1)` in the formula rather than hard-coding "2023" so the allocation automatically uses the prior year without script changes each planning cycle.

---

### Variant 5 — Reference Dimension

Adds a custom dimension (beyond the default Date) as a reference dimension, so allocation weights are calculated **independently for each member** of that dimension rather than pooled together.

**When to use**: Source data is stored against multiple members of a dimension (e.g. multiple audit trails) and you want each member's pool to be allocated using its own driver weights — not a combined pool.

**Allocation Step configuration:**
- Source Context: standard filters
- **Driver Context → Reference Dimension**: add the additional dimension (e.g. **Audit**) alongside the default Date
- Keep Source: unchecked
- Booking Account: optional

**Allocation Rule columns:**
```
Entity (Source) | Account (Driver) | Entity (Target)
Cost Collector  | Headcount        | All Portfolios
```

Note: No audit column in the rule — the Reference Dimension handles the per-audit-trail logic automatically.

**How it works — worked example:**

```
Source data (EXP1, January):
  CC / Input  audit: $62,500
  CC / Input2 audit: $100,000

Driver data (HC, January):
  P1_L / Input  audit: 10
  P1_L / Input2 audit: 20
  (Total Input HC: 100, Total Input2 HC: 100)

With Reference Dimension = Audit:
  P1_L/Input receives:  $62,500  × 10/100 = $6,250   (uses Input pool only)
  P1_L/Input2 receives: $100,000 × 20/100 = $20,000  (uses Input2 pool only)

Without Reference Dimension:
  P1_L HC = 10 + 20 = 30, Total HC = 200, weight = 30/200 = 0.15
  P1_L/Input receives:  $62,500  × 0.15 = $9,375     (WRONG — mixes pools)
  P1_L/Input2 receives: $100,000 × 0.15 = $15,000    (WRONG — mixes pools)
```

**Key insight**: Without Reference Dimension, SAC collapses all audit trail HC values together before calculating weights. With Reference Dimension = Audit, each audit trail's costs are allocated using only that audit trail's driver values.

> **Default Reference Dimension**: Date (Year, Quarter, Month) is **always pre-populated** as a reference dimension. This means January's $62,500 is allocated using January driver weights, February's using February weights, etc. You almost never need to remove this.

---

## Running Allocations from a Story

Allocations are executed from a **Data Action Starter** widget in a story — not from the data action editor directly.

**Data Action Starter configuration:**
- **Label**: display name shown on the button
- **Data Actions**: link to the DA containing allocation steps
- **Always run in background**: checkbox (useful for long-running multi-step allocations)
- **Default strategy**: typically "Automatic Planning Area"
- **Publish to target version after execution**: controls whether the planning buffer is published
- **Parameters**: exposes DA parameters (e.g. Target Version, YearMonth) as prompts to the user

**Run scope options** (presented at execution time):
1. **Use Point of View** — runs using the current story filter context
2. **Refine Filters** — opens a dialog to set specific dimension member filters before running
3. **No Filters** — runs across the full model scope (use with caution)

---

## Common Gotchas

| Gotcha | Explanation | Fix |
|---|---|---|
| Source not zeroing out | Keep Source is checked | Uncheck Keep Source if source should be zero after allocation |
| Double-counting on re-run | Overwrite Target not checked | Check Overwrite Target so existing results are cleared before re-run |
| All targets get same % regardless of period | Date not set as Reference Dimension | Date is default — verify it appears in Driver Context Reference Dimension field |
| Allocation distributes from wrong audit trail | No Overwrite Dimension set | Add Overwrite Dimension = Audit and specify the source audit member in the rule |
| Driver reads 0 for all targets | Driver account is calculated but LOOKUP version is wrong | Check the LOOKUP formula — version string must match exactly including "public." prefix |
| Reciprocal allocation doesn't fully settle | Number of Repetitions = 1 | Increase repetitions (test with 7–10 for reciprocal scenarios) |
| Results land in wrong account | Booking Account not set / set to wrong account | Check Target Context → Booking Account field |
| Allocation runs but no data in Allocs version | Results version filter in story doesn't include Allocs | Add "public.Allocs" to the Version filter in the story table |

---

## Allocation Step vs Advanced Formula Step

| Capability | Allocation Step | Advanced Formula Step |
|---|---|---|
| Proportional distribution | ✅ Native — driver-based math built in | Manual — must code the ratio calculation |
| Multiple senders/receivers | ✅ Multiple rule rows | Requires FOREACH loop |
| Reciprocal allocations | ✅ Repetitions field | Not directly supported |
| Cross-version driver | ✅ Calculated account member | RESULTLOOKUP with version filter |
| Complex conditional logic | ❌ Not supported | ✅ IF/ELSEIF/FOREACH |
| Write to multiple accounts | ❌ One booking account per step | ✅ Multiple DATA() statements |

**Rule of thumb**: Use Allocation Step for proportional cost distribution. Use Advanced Formula for anything requiring conditional logic, iteration, or calculations that don't fit the proportional model.

---

## Useful Links

- [SAP Community: Allocation Blog Part 1 — Basic](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-analytics-cloud-allocation-part-1-basic-allocation/ba-p/)
- [SAP Community: Allocation Blog Part 5 — Reference Dimension](https://community.sap.com/t5/technology-blog-posts-by-sap/sap-analytics-cloud-allocation-part-5-allocation-using-a-reference/ba-p/13473875)
- [SAP Learning: Creating Data Actions for Allocations](https://learning.sap.com/courses/leveraging-advanced-features-in-sap-analytics-cloud-for-planning/creating-data-action-allocations)
- [SAP Learning: Performing Allocations](https://learning.sap.com/courses/leveraging-advanced-features-in-sap-analytics-cloud-for-planning/performing-allocations)
