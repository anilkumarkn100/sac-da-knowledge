# Multi-Actions — Reference

**Source**: SAP Help — SAP Analytics Cloud Multi-Actions
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## What is a Multi-Action?

A Multi-Action is a container that orchestrates a **sequence of different step types** into a single executable planning workflow. While a regular Data Action can only contain Copy, Advanced Formula, Allocation, Currency Conversion, and Embedded Data Action steps, a Multi-Action adds:

- **Version Management** steps (copy, publish, reset, delete versions)
- **Data Locking** steps (lock/unlock data slices)
- **API steps** (call external HTTP endpoints)
- **Embedded Data Action** steps (call regular data actions)

Multi-Actions are used to build end-to-end planning workflows that involve multiple operations across versions, locking, and external systems.

---

## Multi-Action vs. Data Action

| Feature | Data Action | Multi-Action |
|---|---|---|
| Copy step | ✅ | ✅ (as Embedded DA) |
| Advanced Formula | ✅ | ✅ (as Embedded DA) |
| Allocation | ✅ | ✅ (as Embedded DA) |
| Currency Conversion | ✅ | ✅ (as Embedded DA) |
| Version Management | ❌ | ✅ |
| Data Locking | ❌ | ✅ |
| API step | ❌ | ✅ |
| Run another Data Action | ✅ (Embedded DA) | ✅ (Embedded DA) |
| Parameters | ✅ | ✅ |
| Conditional branching | Limited | ✅ (Check steps) |
| Rollback on failure | Full rollback | Step-level (configurable) |

---

## Step Types in Multi-Actions

### Embedded Data Action
Runs a regular data action as one step of the Multi-Action.  
- Receives parameters from the parent Multi-Action  
- If it fails, the Multi-Action can continue or abort based on error handling configuration

### Version Management
See `references/version-management.md` for full details.  
Operations: Copy, Publish, Reset, Delete

### Data Locking
See `references/data-locking.md` for full details.  
Operations: Lock, Unlock, Check Lock Status

### API Step
Calls an external HTTP endpoint (REST API).

| Field | Description |
|---|---|
| **URL** | Target endpoint URL |
| **Method** | GET / POST / PUT / PATCH / DELETE |
| **Headers** | Key-value pairs (e.g. Authorization) |
| **Body** | Request payload (JSON) |
| **Response mapping** | Map response fields to parameters for downstream steps |

Use cases: trigger an SAP BTP workflow, call an S/4HANA posting API, notify an external system.

---

## Multi-Action Structure

A typical planning close workflow:

```
Multi-Action: Period Close and Budget Seed
├── Step 1: Data Locking — Lock [Actual][YYYY.MM]
├── Step 2: Embedded DA — Validate actuals (Advanced Formula checks)
├── Step 3: Version Management — Copy [Actual] → [Budget_Seed]
├── Step 4: Embedded DA — Apply growth rates (Advanced Formula)
├── Step 5: Data Locking — Unlock [Budget][YYYY.MM]
├── Step 6: Version Management — Publish [Budget_Seed] → [Budget]
└── Step 7: API Step — Notify workflow system
```

---

## Parameters in Multi-Actions

Multi-Action parameters work the same as Data Action parameters.  
Parameters can be passed to embedded Data Action steps.

```
Multi-Action parameters:
- PlanningPeriod (Member — time dimension)
- GrowthRate (Number)
- TargetVersion (Member — version dimension)

Passed to Embedded DA step:
- DA Parameter "Period" ← Multi-Action PARAM.PlanningPeriod
- DA Parameter "Rate"   ← Multi-Action PARAM.GrowthRate
```

---

## Error Handling

Each step in a Multi-Action can be configured with:

| Setting | Behaviour |
|---|---|
| **Abort on error** | If this step fails, stop the Multi-Action and roll back |
| **Continue on error** | If this step fails, log the error and continue to next step |

**Recommendation**: Use "Abort on error" for steps that change data; use "Continue on error" for informational or notification steps.

### Rollback behaviour
- Embedded DA steps: full rollback of that DA's changes if it fails  
- Version Management and Data Locking steps: these are **not rolled back** on Multi-Action failure — they are committed as they execute
- This means a failed Multi-Action can leave versions locked or partially copied — always design with compensating unlock/reset steps if needed

---

## Triggering Multi-Actions

Multi-Actions can be triggered:
1. **From a Story** — Add a trigger widget or button, map to Multi-Action
2. **Manually** — From the Data Actions / Multi-Actions list in the Planning area
3. **Via Schedule** — SAC Calendar scheduling (if configured)
4. **Via API** — SAC Planning API (same as Data Actions)

---

## Multi-Action Design Patterns

### Pattern 1: Period Close Workflow
```
Lock prior period actuals
→ Validate data quality
→ Publish approved plan
→ Seed next period forecast
→ Unlock new period for planning
```

### Pattern 2: Top-Down Budget Distribution
```
Receive high-level budget input (story input)
→ Validate totals balance
→ Allocate to cost centres (Embedded DA)
→ Allocate to accounts (Embedded DA)
→ Publish to planning version
→ Notify stakeholders (API step)
```

### Pattern 3: Version Comparison Reset
```
Unlock working version
→ Reset working version (clear all data)
→ Copy actuals to working version
→ Apply adjustments (Embedded DA)
→ Lock working version
```

---

## Common Multi-Action Gotchas

### Gotcha 1: Version Management step changes persist even on failure
If step 3 copies a version and step 5 fails, the copy from step 3 is **not rolled back**.  
**Fix**: Structure Multi-Actions so irreversible steps come last, after all validation.

### Gotcha 2: API step authentication fails silently
An API step with an expired token logs the error but the Multi-Action may continue.  
**Fix**: Put the API step early with "Abort on error" to catch auth failures before data changes.

### Gotcha 3: Embedded DA runs with wrong version context
The Embedded DA step does not inherit the Multi-Action's parameter values automatically — they must be explicitly mapped.  
**Fix**: On the Embedded DA step, explicitly map each parameter.

### Gotcha 4: Data lock left open after failed run
If the Multi-Action aborts before the final unlock step, the lock remains.  
**Fix**: Always have a recovery Multi-Action (just the unlock step) ready to run manually.

### Gotcha 5: Sequence order matters for dependencies
If Step 3 depends on data produced by Step 2, but Step 2 is configured "Continue on error" and fails, Step 3 may run on incomplete data.  
**Fix**: Use "Abort on error" for any step whose output is required by a later step.
