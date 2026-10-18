# Data Locking — Reference

**Source**: SAP Help — SAP Analytics Cloud Data Locking
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## What is Data Locking?

Data locking prevents write operations on a specific slice of planning data. A locked slice cannot be modified by data actions, manual story input, or imports until it is unlocked.

Locking is used to:
- Freeze actuals after period close
- Lock approved budget figures
- Protect baseline scenarios from overwrite
- Control planning workflow (only unlock when in the correct planning stage)

---

## Lock Granularity

A data lock is defined by a **data slice** — a combination of dimension members that identifies the locked data.

Typical lock dimensions:
- Version (e.g. lock `Actual`)
- Time period (e.g. lock all months up to and including the close period)
- Entity / Cost Centre / Business Unit

Locking at a parent member level automatically locks all **base members** beneath it.

---

## Lock Types

| Lock Type | Description |
|---|---|
| **Read-only lock** | Data cannot be modified; it can still be read |
| **Version lock** | The entire version is set to read-only at model configuration level |
| **Data slice lock** | Specific intersection of dimension members is locked |

### Version-level lock (model setting)
Set at the model level — affects all users. Controlled in the model's version configuration.  
Actuals versions are often permanently locked this way.

### Data slice lock (Data Locking step)
Dynamic lock applied via a Multi-Action's Data Locking step. Can be applied and removed as part of a planning workflow.

---

## Data Locking Step (Multi-Action)

The Data Locking step is only available in **Multi-Actions**.

### Operations

| Operation | Description |
|---|---|
| **Lock** | Lock the specified data slice |
| **Unlock** | Remove the lock from the specified data slice |
| **Check Lock Status** | Verify whether a slice is locked (used for conditional logic) |

### Configuration fields

| Field | Description |
|---|---|
| **Operation** | Lock / Unlock / Check |
| **Version** | Which version to lock/unlock |
| **Dimension filters** | Dimensions and members defining the lock scope |
| **Time period** | The time range to lock/unlock |

---

## Workflow Integration

Data locking is most useful in a **workflow-driven Multi-Action**:

```
Multi-Action sequence:
1. Validate input data (Advanced Formula check step)
2. Lock the prior period actuals (Data Locking — Lock step)
3. Seed the new forecast (Version Management — Copy step)
4. Unlock the new forecast period (Data Locking — Unlock step)
5. Notify users (optional — API step)
```

---

## Lock Behaviour in Data Actions

When a data action step attempts to **write to a locked slice**:
- The write is silently ignored (no error, no data written)
- This produces **blank output** even though the formula is correct
- The data action does not fail or show an error message

This is the most common reason data actions produce correct-looking but empty results in controlled planning environments.

### How to detect a lock issue
1. Run the data action — result appears blank
2. Check model settings: verify if the target version is locked at model level
3. Check data locking history: look for active locks on the target slice
4. Try writing to a test version that is definitely unlocked — if data appears, the original target was locked

---

## Locking and the Planning Workflow

### Best practice: explicit lock/unlock in Multi-Action
Rather than relying on model-level version locks, manage locks dynamically:
1. Lock approved periods at close
2. Unlock the current planning period when planning opens
3. Re-lock at period close

This gives auditability and controllability without model configuration changes.

### Who can lock/unlock?
Locking permissions are governed by **SAC roles and data access control**:
- Typically a planning administrator or process owner
- Standard planners cannot lock/unlock unless explicitly granted permission
- Multi-Actions that include lock/unlock steps run under the triggering user's permissions

---

## Common Data Locking Gotchas

### Gotcha 1: Data action writes silently fail on locked slice
**Symptom**: Formula appears correct, no error shown, data is blank  
**Cause**: Target slice is locked  
**Fix**: Unlock the target slice before running the data action, or restructure the Multi-Action to unlock, write, then re-lock

### Gotcha 2: Lock scope too broad — unintended lock
**Symptom**: Other planning processes fail after running a lock step  
**Cause**: Lock dimension filter was at a parent level, locking more members than intended  
**Fix**: Always lock at the most granular level required; verify scope before running in production

### Gotcha 3: Unlock step failed, leaving data unlocked
**Symptom**: Approved data is editable when it should be locked  
**Cause**: Multi-Action completed steps 1-N but the final lock step failed  
**Fix**: Add error handling logic; consider a separate Multi-Action just for the lock step so it can be re-run independently

### Gotcha 4: Version lock vs. data slice lock confusion
**Symptom**: Unlock step in Multi-Action has no effect  
**Cause**: The version is locked at the **model level** (version configuration), not via a data slice lock  
**Fix**: Model-level version locks must be changed in the model configuration by an admin — they cannot be unlocked by a data locking step

---

## Checking Lock Status

Before running a data action that writes to a potentially locked slice, verify status:

**In SAC UI:**  
Model → Data Management → Data Locking → View active locks

**Via Multi-Action Check step:**  
Add a "Check Lock Status" step and branch logic accordingly (available in newer SAC versions)

**Diagnostic approach:**  
Attempt a small manual write in the story table — if the cell is greyed out or shows a lock icon, the slice is locked.
