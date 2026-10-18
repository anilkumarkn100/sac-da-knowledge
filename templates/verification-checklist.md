# Data Action Verification Checklist

**Purpose**: Use this after running a data action in SAC to verify the output is correct before sign-off.

---

## Step 1 — Export the Result

1. In SAC, open the story/table connected to the target version
2. Filter to the exact scope the data action targeted
3. Export the table as CSV (right-click → Export)
4. Attach the CSV to Claude with the message: "Please verify this against the expected output"

---

## Step 2 — Pre-Verification Checks (do these before exporting)

- [ ] Data action ran without errors (check status in Data Action Designer)
- [ ] Correct version selected in the story
- [ ] Correct date range / entity filter applied
- [ ] No active data locks on target

---

## Step 3 — Claude Verification

Claude will:
1. Compare exported values to expected values from the requirement
2. Flag any discrepancies
3. Diagnose root cause if values are missing or wrong
4. Suggest corrective action

---

## Step 4 — Sign-Off

| Check | Result | Notes |
|---|---|---|
| All expected cells populated | Pass / Fail | |
| Values match expected amounts | Pass / Fail | |
| No unexpected cells populated | Pass / Fail | |
| Performance acceptable | Pass / Fail | |
| Tested in QA before production | Yes / No | |

---

## Sign-off

| Role | Name | Date |
|---|---|---|
| Developer | | |
| Reviewer | | |
