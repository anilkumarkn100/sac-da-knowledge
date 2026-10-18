# Advanced Formula — Performance Best Practices

**Source**: SAP Help Portal — SAP Analytics Cloud Q3 2026 (2026.15)
**Last updated**: 2026-10-18

---

## Why Performance Matters in Data Actions

Data actions run against a planning buffer. A slow formula with a large scope can:
- Time out during interactive story execution
- Block other users writing to the same model
- Take minutes when it should take seconds

Most performance problems come from an unnecessarily large calculation scope. Estimate your scope before you write:

```
Scope = (# members in each unscoped dimension) × (# members in each other dimension)

Example: 500 accounts × 2,000 cost centres × 48 months × 10 entities = 480 million cells
Add MEMBERSET to restrict to 5 accounts × 10 cost centres × 12 months × 3 entities = 1,800 cells
```

---

## Rule 1: Prefer DATA over FOREACH

`DATA` runs as a set-based batch operation. `FOREACH` runs one iteration per member.

```
// SLOW — FOREACH iterates 200 product members one at a time
FOREACH [d/Product]
  DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue")
                                - RESULTLOOKUP([d/Account] = "Cost")
ENDFOR

// FAST — DATA runs the same calculation for all products in one pass
DATA([d/Account] = "Margin") = RESULTLOOKUP([d/Account] = "Revenue")
                              - RESULTLOOKUP([d/Account] = "Cost")
```

**When FOREACH is actually required**: carry-forward calculations where one iteration depends on the result of the previous one.

```
// FOREACH required — each month's Opening = previous month's Closing
FOREACH [d/Date]
  DATA([d/Movement] = "OPENING") = RESULTLOOKUP([d/Movement] = "CLOSING",
                                                [d/Date] = PREVIOUS())
ENDFOR
```

---

## Rule 2: Use FOREACH.BOOKED to Skip Empty Cells

`FOREACH` iterates all leaf members regardless of whether they have data. `FOREACH.BOOKED` skips members with no booked data.

```
// May iterate 200 products even if only 15 have values
FOREACH [d/Product]
  ...
ENDFOR

// Only iterates products with actual data — potentially 15 instead of 200
FOREACH.BOOKED [d/Product]
  ...
ENDFOR
```

Use `FOREACH.BOOKED` whenever you're reading from an existing data set and don't need to process empty members.

---

## Rule 3: Narrow Scope with MEMBERSET Aggressively

Every dimension without a MEMBERSET restriction is fully iterated. Add MEMBERSET for every dimension whose full scope you don't need.

```
// Without MEMBERSET: 500 accounts × 48 months × 10 entities = 240,000 cells
DATA([d/Version] = "Budget") = RESULTLOOKUP([d/Version] = "Actual") * 1.05

// With MEMBERSET: 3 accounts × 12 months × 1 entity = 36 cells
MEMBERSET [d/Account] = ("Revenue", "Cost", "Headcount")
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Entity] = "Germany"
DATA([d/Version] = "Budget") = RESULTLOOKUP([d/Version] = "Actual") * 1.05
```

---

## Rule 4: Avoid Large ELSE Scope

The ELSE clause inherits whatever scope is left after the IF and ELSEIF conditions. This can be enormous.

```
// BAD — ELSE runs for ALL non-Revenue accounts (potentially thousands)
IF [d/Account] = "Revenue" THEN
  DATA() = 100
ELSE
  DATA() = 200  // runs for all other accounts
ENDIF

// BETTER — explicit MEMBERSET before the DATA in the ELSE path
IF [d/Account] = "Revenue" THEN
  DATA() = 100
ENDIF
MEMBERSET [d/Account] = "Cost"
DATA() = 200
```

---

## Rule 5: Cache Repeated RESULTLOOKUP Calls

Each RESULTLOOKUP triggers a read. If the same value is read multiple times, cache it in a VARIABLEMEMBER.

```
// SLOW — reads Revenue three times
DATA([d/Account] = "Profit") = RESULTLOOKUP([d/Account] = "Revenue")
                              - RESULTLOOKUP([d/Account] = "Cost")
DATA([d/Account] = "Margin%") = (RESULTLOOKUP([d/Account] = "Revenue")
                              - RESULTLOOKUP([d/Account] = "Cost"))
                              / RESULTLOOKUP([d/Account] = "Revenue")
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = "Revenue") * 0.05

// FAST — reads Revenue once, stores in virtual member
VARIABLEMEMBER #rev OF [d/Account]
DATA([d/Account] = #rev) = RESULTLOOKUP([d/Account] = "Revenue")
DATA([d/Account] = "Profit") = RESULTLOOKUP([d/Account] = #rev) - RESULTLOOKUP([d/Account] = "Cost")
DATA([d/Account] = "Margin%") = (RESULTLOOKUP([d/Account] = #rev) - RESULTLOOKUP([d/Account] = "Cost"))
                              / RESULTLOOKUP([d/Account] = #rev)
DATA([d/Account] = "Bonus") = RESULTLOOKUP([d/Account] = #rev) * 0.05
```

---

## Rule 6: Avoid NULL Comparisons

NULL checks are slower than numeric comparisons.

```
// SLOWER
IF RESULTLOOKUP([d/Account] = "Revenue") != NULL THEN

// FASTER
IF RESULTLOOKUP([d/Account] = "Revenue") <> 0 THEN
```

---

## Rule 7: Split Large Scripts into Multiple Steps

A single huge step is harder to debug and may hit timeout limits. Split by business logic:
- Step 1: Seed / Copy base data
- Step 2: Apply growth rates
- Step 3: Calculate derived accounts (margin, ratios)
- Step 4: Allocate to cost centres

This also makes each step independently traceable and re-runnable.

---

## Rule 8: Use CONFIG.GENERATE_UNBOOKED_DATA = OFF (default) Unless You Need Zeroes

When `ON`, SAC writes 0 to every cell in scope where the source is empty. This massively increases write volume.

Only use `ON` when you explicitly need to zero out stale data from a previous run.

```
// If you need to clear stale data, be explicit about scope first
MEMBERSET [d/Version] = "Budget"
MEMBERSET [d/Date] = BASEMEMBER([d/Date].[h/YQM], "2025")
MEMBERSET [d/Account] = ("Revenue", "Cost")  // limit to the accounts you're resetting
CONFIG.GENERATE_UNBOOKED_DATA = ON
DATA() = 0
```

---

## Performance Checklist Before Running

- [ ] Is every dimension scoped via MEMBERSET (or do I intentionally need the full dimension)?
- [ ] Am I using DATA instead of FOREACH where the calculation is independent per member?
- [ ] Am I using FOREACH.BOOKED where I only need to process existing data?
- [ ] Is CONFIG.GENERATE_UNBOOKED_DATA really needed here?
- [ ] Have I cached repeated RESULTLOOKUP calls into VARIABLEMEMBER?
- [ ] Is my ELSE clause accidentally covering a huge member range?
- [ ] Have I estimated the total cell count? (`scope dimension 1 × dim 2 × ... × dim n`)

---

## Using Tracing to Diagnose Performance

If a formula is slow, use the tracing tool to check the actual runtime Calculation Scope:

1. Add a tracepoint at the first DATA() line
2. Run in trace mode
3. In the Watch Area, check **Calculation Scope** — each dimension shows either a single member ID or a count of members
4. Click any count to expand and see which members are in scope
5. If a dimension has 2,000 members when you expected 12, add MEMBERSET for that dimension

See `data-action-tracing.md` for the full tracing workflow.
