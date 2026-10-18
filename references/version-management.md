# Version Management — Reference

**Source**: SAP Help — SAP Analytics Cloud Planning Versions
**Last updated**: 2026-10-18
**SAC version**: 2025.19 (applies to all current SaaS tenants)

---

## Version Types

SAC planning models use **versions** to separate different planning scenarios, actuals, and forecasts.

| Version Type | Description | Writable | Notes |
|---|---|---|---|
| **Actual** | Historical actuals imported from source systems | Usually read-only | Set at model level; often locked by default |
| **Budget** | Annual operating plan | Yes (if unlocked) | Primary target for planning data actions |
| **Forecast** | Rolling forecast | Yes | Multiple forecast versions common (e.g. FC1, FC2) |
| **Plan** | Generic planning version | Yes | Custom label — same behaviour as Budget |
| **Public** | Shared version visible to all users | Depends on lock | Created from private via publish |
| **Private** | User-specific sandbox version | Yes | Not visible to others until published |

---

## Version Dimension

Versions are represented as members of the built-in **Version** dimension (`[d/Version]`).

### Member reference syntax
```
[d/Version].[Actual]        // Specific version by technical ID
[d/Version].[Budget]
[d/Version].[Forecast_FC1]
```

### Checking if a version exists in context
If a data action does not have the target version in its filter context, writes to that version produce **no output** (silent failure). Always verify:
1. The data action's version filter includes all versions referenced in the formula
2. The calling story page has the relevant version in its filter

---

## Version Scope in Data Actions

### Setting version context
Version context is established by:
1. The **story filter** when the data action is triggered from a story
2. The **data action's own filter** (set in the data action's filter configuration)
3. Explicit `MEMBERSET` in an Advanced Formula step

### Version filter configuration
In the data action's configuration panel, the Filters section controls which versions are in scope. Best practice: **explicitly list all versions** the data action reads from or writes to.

If a version is not in the filter:
- Reads from that version return NULL/blank
- Writes to that version are silently ignored

### Explicit version scope in formula
```
// Restrict this block to Budget version only
MEMBERSET([d/Version], {[d/Version].[Budget]});
{
  [d/Measures].[Revenue] = PARAM.BudgetValue;
}
```

---

## Version Management Step (Multi-Action)

The Version Management step type is only available in **Multi-Actions**. It performs version-level operations.

### Operations available

| Operation | Description |
|---|---|
| **Publish** | Promote a private version to public |
| **Copy Version** | Copy all data from one version to another |
| **Reset Version** | Delete all data in a version (returns to empty) |
| **Delete Version** | Remove a version from the model entirely |

### Configuration
- Select the operation
- Specify source version (for Copy)
- Specify target version
- Optionally filter by time period (for partial copy/reset)

### Notes
- Copy Version is useful for seeding a forecast from actuals or copying last year's budget as a starting point
- Reset Version clears **all** data — use with extreme caution; there is no undo
- Publish moves data from a private to public version — users must have appropriate permissions

---

## Private vs. Public Versions

### Private versions
- Created per-user as a personal sandbox
- Not visible to other users
- Data is writable by the owning user
- Identified by a `private_` prefix or user-specific naming convention

### Publishing a private version
Publishing copies data from a user's private version into the corresponding public version. This is typically the final step after review in a planning workflow.

### Version management in Multi-Actions
A common workflow pattern:
1. Seed private version from actuals (Copy step or Version Management Copy)
2. Apply planning adjustments (Advanced Formula steps)
3. Review in story
4. Publish private → public (Version Management Publish step)

---

## Common Version-Related Data Action Issues

### Issue: Blank output when writing to a version
**Cause**: Version not in data action filter context
**Fix**: Add the target version explicitly to the data action's Filters panel

### Issue: Writing to Actual version fails
**Cause**: Actual version is set to read-only at model level
**Fix**: Change version type or use a separate planning version for actuals-based calculations

### Issue: Copy version produces unexpected results
**Cause**: Time period granularity mismatch between source and target
**Fix**: Ensure both source and target use the same time granularity (month vs. quarter vs. year)

### Issue: Reset version wipes more data than expected
**Cause**: Reset scope is too broad (no time filter applied)
**Fix**: Always apply a time period filter when resetting a version

---

## Version Naming Conventions (Recommended)

| Purpose | Convention | Example |
|---|---|---|
| Budget year | `Budget_YYYY` | `Budget_2025` |
| Forecast cycle | `Forecast_FCNN_YYYY` | `Forecast_FC1_2025` |
| Actuals | `Actual` | `Actual` |
| Working copy | `WIP_[descriptor]` | `WIP_Headcount_Apr` |
| Scenario | `Scenario_[name]` | `Scenario_OptimisticCase` |

Consistent naming makes version filters in data actions unambiguous and reduces errors.
