# Blank Output — Diagnostic Checklist

**Purpose**: When a data action runs successfully but produces no data, work through this checklist in order.

---

## Diagnostic Order

### 1. Version Filter
- [ ] Does the source version actually contain data in the specified scope?
- [ ] Is the version name/ID spelled exactly as it appears in the model?
- [ ] Is the target version different from the source version?

### 2. Member Scope
- [ ] Are filters pointing to **leaf members** where data is physically stored?
- [ ] Are you filtering on a parent member expecting data to exist there? (It won't — SAC stores at leaf level)
- [ ] Are member IDs used (not descriptions)?

### 3. Date Granularity
- [ ] What granularity is data stored at in this model? (Year / Quarter / Month / Day)
- [ ] Does the data action scope match that granularity?
- [ ] Example: data stored at Month but step targets Year → blank output

### 4. Dimension Coverage
- [ ] Are ALL dimensions either in scope or explicitly excluded?
- [ ] Any dimension not in scope defaults to base member — does base member have data?

### 5. Account Type
- [ ] Are source and target accounts the same type (INC vs EXP)?
- [ ] Sign convention mismatch can cause values to cancel to zero

### 6. Target Lock State
- [ ] Is the target version locked?
- [ ] Is the target data slice in Restricted or Locked state?

### 7. Source Data Existence
- [ ] Export a small slice from the source version and confirm data exists
- [ ] Check Data Management → Data Preview in the model

---

## Patterns (add entries below as discovered)

_Add entries here linking to specific SAP Community threads or team discoveries._
