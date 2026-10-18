# SAC Data Action Assistant

## Description
Use this skill when a developer needs help building, debugging, or verifying an SAP Analytics Cloud (SAC) Data Action. Covers requirement intake, data action spec generation, context validation, and output verification.

---

## When to Use This Skill

- Developer describes a planning calculation requirement
- Developer is getting blank output from a data action
- Developer needs to verify data action output is correct
- Developer is unsure how to scope a copy, allocation, or formula step
- Team wants to validate a data action before deploying to production

---

## Commands

| Command | What it does |
|---|---|
| `/da-help` | Show how to use this skill — what to provide, how to ask, how to generate data exports |
| `/da-context` | Start Phase 1 — provide model context |
| `/da-build` | Start Phase 2 — build a data action or allocation spec |
| `/da-verify` | Start Phase 3 — verify actual vs expected output |
| `/da-allocate` | Allocation-specific intake — starts the allocation variant decision tree |

---

## `/da-help` — How to Use This Skill

### What this skill does
This skill guides you through building, debugging, and verifying SAC Data Actions — including advanced formula steps, allocation steps, copy steps, and multi-action sequences.

### The three things you can ask

**1. Build a new data action**
Describe what you want to calculate. Be specific:
- What should move where? (source cost center → target cost centers)
- What drives the distribution? (headcount, floor area, etc.)
- Which version holds source data? Which version receives results?
- What should happen to the source after the calculation?

> Example: "I want to allocate HR costs from cost center CC_HR to all operating units based on headcount. Source is public.Plan. Results should go to public.Allocs. Source should zero out."

**2. Debug a blank or wrong result**
Describe what you ran and what you expected vs what you got. Attach a CSV export of the target slice after the run.

> Example: "I ran my allocation and the target slice shows zeros. Source also shows zeros so it ran. Attaching CSV."

**3. Ask a question about allocation or formula behaviour**
Ask directly. No command needed.

> Example: "What does Keep Source do in an allocation step?" or "How do I allocate from a prior-year headcount?"

---

### How to generate the model data export (for `/da-context`)

Claude needs to see real member IDs, not just descriptions. The fastest way to provide model context is to export a small data slice as CSV from a SAC story.

**Steps to generate the CSV:**

1. Open your SAC story that uses the planning model you want to build a data action for
2. Add a new table widget (or use an existing one)
3. Configure the table to show:
   - All dimensions as rows or columns (Entity, Cost Center, Account, Date, Version, Audit — whatever your model has)
   - Include at least a small slice of **booked data** (e.g. one version, one quarter, one cost center)
   - Make sure the table shows **member IDs** not just descriptions: in table settings, set "Display" → "ID" for each dimension
4. Right-click the table → **Export** → **CSV**
5. Attach the downloaded CSV to your message here

**What Claude reads from the CSV:**
- Exact dimension names as used in the model (these must match formula syntax exactly)
- Member IDs for accounts, cost centers, entities, versions (these go directly into scope filters)
- Data granularity — e.g. is Date stored at Month level (Jan 2025) or Quarter (Q1 2025)?
- Which members have booked data vs which are empty

**Minimum useful export**: 1 version × 3–5 accounts × 3–5 cost centers or entities × 3 months. Roughly 50–200 rows is ideal.

**Alternative if you can't export a CSV:**
Paste a copy of your dimension list from the model editor, including:
- Dimension name, type (Account/Date/Generic/Organisation), hierarchy name
- A few representative member IDs for each (especially Account and the main org dimension)
- Which version(s) have data

---

## Three-Phase Workflow

### Phase 1 — Context Intake (`/da-context`)
Before generating anything, Claude must understand the model.

**Required inputs (ask for these if not provided):**
1. Model name and type (Planning Model confirmed?)
2. Dimensions list with types (Account, Version, Date, Generic, Organisation)
3. Data granularity per dimension (e.g. Date stored at Month level)
4. Which versions exist and which have booked data
5. A sample data export CSV from the model (preferred over manual description)

**If developer attaches a CSV export:**
- Read the CSV to derive dimension names, member IDs, granularity, and which cells are booked
- Build a model context profile and confirm it with the developer before proceeding

**Always read:** `references/context-and-scope.md` before completing this phase.

---

### Phase 2 — Build (`/da-build`)
Generate the data action specification.

**Steps:**
1. Read `references/data-action-steps.md` — confirm correct step type
2. Read `references/advanced-formulas.md` — if formula step required
3. Search `patterns/` for any known issues matching the requirement
4. For allocation requirements: read `references/allocations.md` and `patterns/allocation-gotchas.md`
5. For version operations: read `references/version-management.md` and `patterns/version-scope-issues.md`
6. For cross-model requirements: read `patterns/cross-model-tips.md`

**Output format:**
Produce a structured data action specification with:
- Step type
- Source scope (version, dimensions, member filters)
- Target scope (version, dimensions, member filters)
- Mappings
- Advanced formula script (if applicable)
- Parameters (name, type, default)
- Pre-run validation checklist
- Expected output description

**Pre-validate before handing over:**
- Source version has booked data in the scope specified?
- Target version is not locked?
- All dimension filters use correct member IDs (not descriptions)?
- Date granularity matches model storage level?
- Account types are consistent (no sign-flip issues)?

---

### Allocation Step — Variant Selection (`/da-allocate`)

When the requirement involves an allocation step, run this intake before building the spec. Ask the developer all questions that are not already answered by the context provided.

**Step 1 — Intake questions (ask any that are unanswered):**

| # | Question | Why it matters |
|---|---|---|
| 1 | Does the source cost center need to **zero out** after allocation, or retain its value? | Keep Source checkbox |
| 2 | Should results land in the **same account** as source, or in a separate named "allocation in" account? | Booking Account (Variant 3) |
| 3 | Is the **driver data** in the same version and year as the allocation target, or a different version/year? | Calculated Account driver (Variant 4) |
| 4 | Does the source have **multiple audit trail entries** (e.g. Input by User + System Generated) that should be handled independently? | Overwrite Dimension (Variant 2) or Reference Dimension (Variant 5) |
| 5 | Are there **two cost centers that allocate to each other** (reciprocal)? | Number of Repetitions > 1 |
| 6 | Will this allocation be **re-run** during the same planning cycle? | Overwrite Target must be on |
| 7 | Should each **audit trail** use its own driver weights, or share a combined pool? | Reference Dimension (Variant 5) |

**Step 2 — Variant decision tree:**

```
Q: Is driver data in a different version or year?
  YES → Variant 4 (Calculated Account driver) + LOOKUP formula
  NO  → continue

Q: Does each audit trail need independent driver weights?
  YES → Variant 5 (Reference Dimension = Audit)
  NO  → continue

Q: Should results land in a separate booking account?
  YES → Variant 3 (Booking Account)
  NO  → continue

Q: Should allocation be restricted to one specific audit trail only?
  YES → Variant 2 (Overwrite Dimension = Audit)
  NO  → Variant 1 (Basic Proportional)

Note: Variants 2–5 can be combined. E.g. Variant 3 + 5 = Booking Account with Reference Dimension.
Two cost centers allocating to each other → any variant + Number of Repetitions ≥ 7.
```

**Step 3 — Spec template for an allocation step:**

```
Allocation Step: [Name]
Variant: [1–5]

SOURCE CONTEXT
  Version filter:   [e.g. public.Plan]
  Account filter:   [e.g. All Expense]
  Entity filter:    [e.g. CC_HR]
  Keep Source:      [ON / OFF]

DRIVER CONTEXT
  Reference Dimensions: Date (default) [+ Audit if Variant 5]
  [If Variant 4: Calculated account member name + LOOKUP formula]

TARGET CONTEXT
  Booking Account:  [account name / empty]
  Overwrite Target: [ON / OFF]

ALLOCATION RULES
  # | Source          | [Overwrite Dim] | Driver       | Target           | [Target Audit]
  1 | CC_HR           | [Input by User] | Headcount    | All Portfolios   | [Allocs]

REPETITIONS: [1 / 7+ for reciprocal]

EXPECTED RESULT
  - Source CC: [zeros out / retains value]
  - Results visible in: public.Allocs version
  - Story filter needed: include public.Allocs in Version filter
```

**Step 4 — Pre-build checklist for allocations:**

- [ ] Source version confirmed to have booked data at the specified scope
- [ ] Driver account has booked data for target members in same scope
- [ ] If Calculated Account driver: formula verified with correct version string (includes "public." prefix)
- [ ] Overwrite Target is ON (unless first run ever and no re-run needed)
- [ ] Date appears in Reference Dimension field
- [ ] If reciprocal: Repetitions set to ≥ 7
- [ ] Story filter will include public.Allocs after run

**Always read** `references/allocations.md` and `patterns/allocation-gotchas.md` before producing an allocation spec.

---

### Phase 3 — Verify (`/da-verify`)
Compare actual output to expected output.

**Steps:**
1. Ask developer to export the target data slice as CSV after running the data action
2. Compare actual values to expected values calculated from the spec
3. Identify any discrepancies
4. Diagnose root cause using `patterns/blank-output-causes.md`
5. Suggest corrective action

---

## Context Failure Diagnosis

If developer reports blank output, immediately check:

1. **Version filter** — is source version correct and does it have data?
2. **Member scope** — are filters pointing to leaf members where data is stored?
3. **Date granularity** — does step scope match model's storage granularity?
4. **Dimension coverage** — are all required dimensions included in scope?
5. **Account type** — any sign convention mismatch?
6. **Target lock state** — is target version or data slice locked?

Read `patterns/blank-output-causes.md` for full diagnostic checklist.

---

## Knowledge Base Reading Order

| Situation | Files to read |
|---|---|
| Any data action | `references/data-action-steps.md` |
| Advanced formula step | `references/advanced-formulas.md` |
| Blank output reported | `patterns/blank-output-causes.md` |
| Copy step issues | `patterns/copy-step-issues.md` |
| Allocation step | `references/allocations.md` + `patterns/allocation-gotchas.md` |
| Multi-action | `references/multi-actions.md` |
| Version confusion | `references/version-management.md` + `patterns/version-scope-issues.md` |
| Cross-model | `patterns/cross-model-tips.md` |
| New SAC feature used | `references/whats-new-latest.md` |

---

## Output Rules

- Always specify member IDs in MDX format: `[Dimension].[Hierarchy].&[MemberID]`
- Always state the SAC version the spec is written for (check `references/whats-new-latest.md` for current)
- Include a pre-run checklist with every spec
- If context is insufficient to safely generate a spec, ask for the missing information rather than guessing
- Flag any assumptions made explicitly

---

## Maintainer
Cognivion SAC Practice
Repo: github.com/[your-org]/sac-da-knowledge
