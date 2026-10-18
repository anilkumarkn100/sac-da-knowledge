# Allocation — Known Gotchas and Patterns

**Sources**: jraymond15 SAP Community blog series (Parts 1–5), SAP Learning course, SAP Help 2026.15
**Last updated**: 2026-10-18

---

## G1 — Source Not Zeroing Out After Allocation

**Symptom**: After running the allocation, the source cost center still shows the original values. Expected it to be zeroed out.

**Cause**: **Keep Source** checkbox is ticked in Source Context → Write Mode.

**Fix**: Uncheck Keep Source. This is the default (unchecked = source zeroes out). Only tick it when the source should retain its value (e.g. allocating from an Actual version reference without changing it).

**When Keep Source = ON is correct**: You are reading from a reference version/snapshot and distributing costs without consuming the source amounts.

---

## G2 — Double-Counting on Re-Run

**Symptom**: Running the allocation a second time adds to previous results instead of replacing them. Values are 2× or 3× expected.

**Cause**: **Overwrite Target** is not checked in Target Context → Write Mode.

**Fix**: Check Overwrite Target. This deletes existing allocation results in the target scope before writing new ones. Always enable this for allocations that may be re-run.

**Note**: Results land in version **public.Allocs** by default. Make sure Overwrite Target clears the Allocs slice, not just the Input slice.

---

## G3 — All Targets Get the Same Percentage Regardless of Period

**Symptom**: January and December targets receive identical allocation percentages even though headcount or driver data differs by month.

**Cause**: **Date is not set as a Reference Dimension** in Driver Context, or the Reference Dimension field was accidentally cleared.

**Fix**: Verify that **Date (Year, Quarter, Month)** appears in the Driver Context → Reference Dimension field. It is pre-populated by default — but if the step was copied or reconfigured it may have been lost. Without it, SAC sums all period driver values together and applies one combined weight across all periods.

---

## G4 — Allocation Distributes from the Wrong Audit Trail

**Symptom**: Allocation is picking up data from "System Generated" or another audit member instead of only "Input by User" records.

**Cause**: **Overwrite Dimension** is not configured. Without it, SAC allocates from all audit trail records combined.

**Fix**: Set Overwrite Dimension = Audit in the step configuration, then add Audit (Overwrite) and Audit (Target) columns to the allocation rule. Specify the exact source audit member (e.g. "Input by User") and the target audit member (e.g. "Allocs") in the rule rows.

---

## G5 — Driver Returns Zero for All Targets — Nothing Allocated

**Symptom**: Allocation runs without error but all target cells are 0. Source is also 0 after run (values were consumed but nothing distributed).

**Causes and checks (in order):**

1. **Driver account has no data in the target scope** — check that the account used as Driver Dimension actually has booked values for the target entities/cost centers in the version/period being allocated
2. **Driver is a calculated account with wrong LOOKUP** — if using a calculated account member as driver, verify the LOOKUP formula references the correct version string (must include "public." prefix, e.g. `"public.Actual"` not `"Actual"`)
3. **Version filter in Source Context excludes driver data** — driver data must be readable in the scope defined by the Source Context filters
4. **Target dimension member is a parent node** — allocation rule target should point to leaf members or a parent that rolls up to the intended receivers; if the driver account has data only at leaf level and the rule targets a parent with no direct data, weights may resolve to 0

---

## G6 — Reciprocal Allocation Leaves Large Residuals

**Symptom**: After running, the two mutually-allocating cost centers (e.g. HR and IT) still show significant non-zero balances instead of zeroing out.

**Cause**: **Number of Repetitions = 1** (default). One pass is not enough to fully settle reciprocal allocations.

**Fix**: Increase Number of Repetitions. Test with progressively higher values (3, 5, 7, 10). The correct value depends on the driver ratios — steeper ratios converge faster; near-equal ratios may need more iterations. Maximum = 10.

**Example**: HR allocates 20% to IT, IT allocates 10% to HR. At Repetitions = 1 there are residuals. At Repetitions = 7 both CCs approach zero.

---

## G7 — Results Not Visible in Story Table

**Symptom**: Allocation runs successfully (no errors in log) but the story table shows no change.

**Cause**: The story table's **Version filter does not include public.Allocs**. Allocation results land in the Allocs audit trail of the target version, which is separate from Input.

**Fix**: In the story table, add `public.Allocs` to the Version dimension filter (or switch the version chip to show All Versions). Results will appear under the Allocs audit trail row.

---

## G8 — Booking Account Results Mixed With Source Costs

**Symptom**: The target entities show allocation amounts added directly to the same accounts as source costs (e.g. "Labor" account shows both original labor cost and allocated-in HR costs combined).

**Cause**: **Booking Account is not configured**. Without it, results post to the same account as the source.

**Fix**: Set a dedicated Booking Account in Target Context (e.g. "HR Allocation In", "Utilities Allocation In"). This keeps allocated amounts in a separate named account so actuals vs plan comparisons remain clean.

**When Booking Account = blank is correct**: Simple pass-through where you want costs to land in the same account on the target CC. Acceptable when the source CC will be fully zeroed and no comparison of original vs allocated is needed.

---

## G9 — Reference Dimension Giving Wrong Weights (Audit Trail Mixing)

**Symptom**: Target allocations look wrong — amounts seem too high for one audit trail and too low for another. Audit trail pools appear to be mixed.

**Cause**: **Audit dimension is not set as a Reference Dimension**. SAC is combining driver values from both audit trails to calculate a single weight, then applying that combined weight to each trail's source independently — producing incorrect results.

**Fix**: Add **Audit** to Driver Context → Reference Dimension (alongside the default Date). This forces SAC to calculate driver weights separately for each audit trail.

**Contrast**:
- Without Audit as ref dim: P1_L HC = 10 (Input) + 20 (Input2) = 30/200 = 15% applied to both trails
- With Audit as ref dim: P1_L Input HC = 10/100 = 10% applied only to Input; P1_L Input2 HC = 20/100 = 20% applied only to Input2

---

## G10 — Calculated Account Driver Uses Wrong Version or Year

**Symptom**: Allocation runs but targets receive unexpected amounts. Driver values appear to come from the wrong period or version.

**Cause**: The **LOOKUP formula in the calculated account member** references a hard-coded year or version that no longer applies.

**Fix options**:
- Use `Previous("Year", 1)` for a dynamic prior-year offset instead of hard-coding `[d/Date]="2023"`
- Verify the version string includes the `"public."` prefix: `[d/Version]="public.Actual"` not `"Actual"`
- Check that the calculated account member is saved and published before the data action runs

**Example of correct dynamic formula**:
```
LOOKUP([Number of People], [d/Date]=Previous("Year",1) AND [d/Version]="public.Actual")
```

---

## G11 — Allocation Step vs Advanced Formula — Wrong Tool for the Job

**Symptom**: Developer trying to write an Advanced Formula step to do proportional cost distribution, or trying to use an Allocation step for conditional/multi-account logic.

**Rule**:
- Use **Allocation Step** for: proportional distribution based on a driver, multi-sender/receiver cost pools, reciprocal allocations
- Use **Advanced Formula** for: conditional allocation (IF/ELSEIF on account type), writing to multiple target accounts in one step, allocation with complex business rules the step UI can't express

**Key constraint**: Allocation step can only write to **one Booking Account** per step. If you need to distribute costs to different accounts based on conditions, use Advanced Formula.

---

## Allocation — Pre-Build Checklist

Before configuring any allocation step, confirm answers to these questions:

- [ ] Which cost center(s) are the **source** (sender)?
- [ ] Which dimension holds the source — Entity or Cost Center or other org dimension?
- [ ] Which **account(s)** hold the costs to be distributed?
- [ ] Which account holds the **driver data** (headcount, kWh, floor area, etc.)?
- [ ] Where does driver data live — same version, or cross-version via calculated account?
- [ ] Which cost center(s) / entities are the **targets** (receivers)?
- [ ] Should the source **zero out** after allocation (Keep Source = off) or retain value (Keep Source = on)?
- [ ] Should results go to the **same account** as source, or a **separate booking account**?
- [ ] Is there more than one **audit trail** in the source that needs to be handled independently? (→ Overwrite Dimension or Reference Dimension)
- [ ] Are any cost centers **mutually allocating** to each other? (→ Repetitions > 1)
- [ ] Will this allocation be **re-run** periodically? (→ Overwrite Target = on)
- [ ] Which **version** holds the source data, and which version receives results?
