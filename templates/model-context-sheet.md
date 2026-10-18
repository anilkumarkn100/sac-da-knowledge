# Model Context Sheet

**Purpose**: Fill this in once per SAC planning model before requesting data action help. This is the minimum context Claude needs to avoid blank output and generate correct specs.

**Tip**: Instead of filling this manually, export a small data slice (1 entity, 1 month, all accounts) from Data Management as CSV and attach it. Claude will derive the context automatically.

---

## 1. Model Details

| Field | Value |
|---|---|
| Model name | |
| Model type | Planning Model / Analytic Model |
| SAC tenant URL | |
| Current SAC version | |

---

## 2. Dimensions

| Dimension Name | Type | Hierarchy Levels | Data Stored At (Leaf Level) | Notes |
|---|---|---|---|---|
| Version | Version | Flat | All members | |
| Date | Date | Year → Quarter → Month | Month | |
| Account | Account | L1 → L5 | L5 (leaf) | |
| | Generic | | | |
| | Organisation | | | |

---

## 3. Version Landscape

| Version ID | Version Name | Type | Has Booked Data? | Date Range with Data | Notes |
|---|---|---|---|---|---|
| | Actual | Public | Yes | | |
| | Budget | Public | No | | Empty — target |
| | Forecast | Public | Partial | | |

---

## 4. Account Structure (summary)

| Account Group | Members in Scope | Type (INC/EXP) | Input-Ready? |
|---|---|---|---|
| Revenue | | INC | Yes |
| COGS | | EXP | Yes |
| | | | |

---

## 5. Known Data Gaps / Sparse Areas

_Describe any dimensions or member combinations where data is known to be missing or sparse._

---

## 6. Data Locks

| Locked? | What is locked | Lock State |
|---|---|---|
| | | |
