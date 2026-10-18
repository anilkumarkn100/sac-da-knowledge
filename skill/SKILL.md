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
