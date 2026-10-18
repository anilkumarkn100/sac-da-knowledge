# Data Action Tracing

**Source**: SAP Help Portal — SAP Analytics Cloud Q3 2026 (2026.15)
**Last updated**: 2026-10-18

---

## What Is Data Action Tracing?

Data action tracing is a built-in debugging tool that lets you **execute a data action and inspect intermediate results at manually specified locations** (tracepoints) without pausing execution. It shows you the data state at each tracepoint and the changes between consecutive tracepoints.

This is the official debugging path for advanced formula steps. Use it when:
- Formula produces no output (blank result)
- Formula writes to the wrong members
- Numbers look off (sign, magnitude, scope)
- You need to verify MEMBERSET scope at runtime
- You want to check variable values mid-execution

---

## Key Concepts

| Term | Meaning |
|---|---|
| **Tracepoint** | A manually placed marker on a line of the script. Data state is captured at this point during trace-mode execution. |
| **Trace mode** | Running a data action via the tracing panel instead of the normal Run button. Changes are written to a **tracing version** — the real version is not touched. |
| **Tracing version** | A temporary version created automatically for the trace run. Discarded after the session. |
| **Watch area** | The panel that shows tracepoint results — parameters, variables, calculation scope, and data changes. |

---

## Tracepoint Limits

| Limit | Default value |
|---|---|
| Maximum tracepoints per data action (including embedded DAs) | **20** |
| Maximum traced iterations per FOREACH or FOR loop | **10** |

These limits are set in tenant configuration. Contact your SAC admin if you hit them regularly.

---

## How to Add Tracepoints

### In Saved Mode (recommended for debugging sessions)
1. Open the data action
2. In the **script editor**, click in the **left margin** next to a line number
3. The tracepoint indicator appears briefly (hover to see it again)
4. Tracepoints are **automatically saved** — no need to save the data action
5. Tracepoints appear as **red filled circles**

### In Edit Mode (while actively editing the script)
1. Click in the left margin at any line — tracepoint not saved until you save the data action
2. Tracepoints appear as **empty circles**
3. After saving, unsupported lines are removed, misplaced tracepoints are moved

### Placement Rules
- A tracepoint applies at the **end of a calculation block** (not the beginning of a line)
- SAC automatically moves a tracepoint placed mid-calculation to the **beginning of the calculation**
- Tracepoints **cannot** be added to statements not supported by tracing (e.g. CONFIG lines, MEMBERSET lines)
- Tracepoints **cannot** be added from the Visual Tool — script editor only
- Tracepoints are **not affected** by undo/redo

### Tracepoints Inside Loops (FOREACH / FOR)

When you click a line inside a loop:
1. A **Conditional Tracepoint** dialog appears
2. For **FOREACH loop**: select the dimension member(s) you want to trace
3. For **FOR loop**: set the variable value(s) you want to trace
4. If the line is inside multiple nested loops, you set conditions for each loop
5. Confirm with **OK** — the condition is saved

If you're in **edit mode** when adding a tracepoint inside a loop, the dialog won't appear. After you save, the tracepoint shows a **Condition Missing** warning. Click the Edit Condition icon in the Tracing panel to set the condition.

### Step-Level Tracepoints (between steps)
You can also add tracepoints **before the first step, after the last step, or between steps** by clicking the connector between steps in the left pane of the data action page.

### Removing a Tracepoint
Click the tracepoint indicator again to remove it. Or hover over it in the Tracing panel and click the Delete Tracepoint icon.

---

## Running a Trace

1. Open the **Tracing Panel** — click the **(Open Tracing Panel)** icon in the toolbar
2. Click **Run** in the tracing panel
3. Set any required parameters in the prompt
4. Click **Run** again

The data action runs in full (all steps execute). Changes go to a **tracing version** — the real target version is not modified.

### Required Permissions
- **Create** + **Update** permissions on the Data Action item to add/delete tracepoints
- **Execute** permission on the data action to run it in trace mode
- **Write** access to dimension members if data access control is enabled

---

## Reading Tracing Results — The Watch Area

After a successful trace run, click any tracepoint in the panel to see its Watch Area:

| Component | What it shows |
|---|---|
| **Parameters** | Parameter names and values selected at run time |
| **Variables** | All FLOAT and INTEGER variable names and their values at this point |
| **Calculation Scope** | The exact scope at run time — includes MEMBERSET scope and condition scope, but NOT statement scope. Single members show their ID; multiple members show a count you can click to expand. |
| **Combined Scope** | Combinations of dimension members for conditions like `IF RESULTLOOKUP(…) THEN`. |
| **Data Changes** | A story-like table showing what values changed between the previous tracepoint and this one |

### Reading the Calculation Scope

The Calculation Scope is the most diagnostic component. It answers:
- Is MEMBERSET actually restricting this dimension to the expected members?
- Is the scope narrower/wider than expected?
- Are the correct versions in scope?

If a dimension shows a large member count (e.g. 2,847 Date members) when you expected 12 months, MEMBERSET is not working as intended.

---

## Tracing vs TRACE() Function

Both are debugging tools but serve different needs:

| | Tracing (tracepoints) | TRACE() function |
|---|---|---|
| Output | Visual table in UI — shows data state, scope, variables | Text in execution log |
| Requires UI setup | Yes — add tracepoints in script editor | No — write inline in formula |
| Works in Multi-Action | Yes | No |
| Best for | Complex scope/calculation debugging | Quick sanity-check during development |
| In trace mode only? | Yes — only captures during trace-mode run | No — always writes to log when run |

```
// TRACE() syntax — still useful for quick checks
TRACE(RESULTLOOKUP([d/Account] = "Revenue"), "Revenue before allocation")
TRACE(@TotalAmount, "Running total after month 3")
```

---

## Diagnostic Workflow Using Tracing

### Problem: Formula produces blank output

1. Add a tracepoint **at the first DATA() line** in the formula
2. Run in trace mode
3. Check **Calculation Scope** — is [d/Version] the expected version? Is [d/Date] scoped correctly?
4. If scope looks wrong: check your MEMBERSET lines — add a tracepoint before and after MEMBERSET to see if it's applying
5. Check **Data Changes** — if no rows show under "changed", nothing was written. Is the scope correct but source data is empty? Try a tracepoint at the RESULTLOOKUP line and check Variables.

### Problem: Wrong numbers

1. Add tracepoints before and after each major calculation block
2. Compare **Data Changes** between tracepoints to isolate where the value goes wrong
3. Check **Variables** panel for intermediate FLOAT/INTEGER values
4. Check CONFIG settings (FLIPPING_SIGN_ACCORDING_ACCTYPE, TIME_HIERARCHY) — these change what RESULTLOOKUP returns

### Problem: Too many cells written (performance issue)

1. Add a tracepoint at the start
2. Check **Calculation Scope** — identify which dimensions have unexpectedly large member counts
3. Add MEMBERSET for those dimensions to narrow scope

---

## Common Mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Adding tracepoint in Visual Tool | Option not available | Switch to Script Editor view |
| Not setting FOREACH condition | Tracepoint shows "Condition Missing" warning | Click Edit Condition, set the member |
| Running trace but checking main version | Trace writes to tracing version, not real version | Check the tracing version in the Watch Area |
| More than 20 tracepoints | Tracepoints beyond limit are ignored | Remove unused tracepoints |
| Tracepoint on CONFIG or MEMBERSET line | SAC removes or rejects it | Move tracepoint to first DATA() or RESULTLOOKUP line |
