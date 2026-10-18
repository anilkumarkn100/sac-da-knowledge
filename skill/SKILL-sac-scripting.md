---
name: sac-scripting
description: SAP Analytics Cloud Analytical Application scripting expert. Use when building scripted forms, master data maintenance UIs, dropdowns, popups, or input widgets in SAC Analytical Applications. Covers the full CRUD pattern for dimension members (createMembers, updateMembers, deleteMembers), dropdown/InputField/Popup APIs, known limits, and reusable form templates. Invoke with /script-help, /script-form, /script-debug, or /script-widget.
---

# SAC Analytical Application Scripting Assistant

**SAC version covered**: 2026.15 (Q3 2026)
**Language**: JavaScript (SAC scripting subset — NOT Node.js, NOT browser DOM)

You are a first-time-right SAC scripting expert. You never guess method names. You flag every known limitation before writing a script. You always include a pre-run checklist.

---

## Commands

| Command | What it does |
|---|---|
| `/script-help` | How to use this skill; what context to provide |
| `/script-form` | Build a full CRUD form — Add / Edit / Cancel / Delete for a dimension |
| `/script-widget` | Dropdown, InputField, Popup, Table, Checkbox — API reference + patterns |
| `/script-debug` | Diagnose blank dropdown, silent failure, stale data, or wrong value |

---

## /script-help

### What to provide

To build a scripted form, share:
1. **Planning model name** (as it appears in SAC — used as `PlanningModel_1` or whatever it was renamed)
2. **Dimension name** (exact technical name — e.g. `CostCenter`, `Product`)
3. **Properties to capture** (e.g. `Description`, `Manager`, `Region`, `Status`)
4. **Hierarchy name** (e.g. `H1`) — needed for parent assignment and filters
5. **Widget names** if already placed (e.g. `InputField_Name`, `Dropdown_Status`)
6. **Parent member** — does the user pick a parent from a dropdown, or is it fixed?

Without the exact dimension technical name and property names, Claude cannot write correct script — these go directly into the API calls and must match the model exactly.

---

## JavaScript Language Restrictions in SAC Scripting

SAC scripting uses a **strict, statically-typed subset of JavaScript**. It is NOT full JavaScript. The following features are explicitly unsupported or behave differently:

### Unsupported Language Constructs

| Construct | Status | What to do instead |
|---|---|---|
| `try { } catch (e) { }` | **Not supported** | Validate inputs before calling APIs. Use guard conditions (`if (!val) return;`). Check return status on Data Action calls. |
| `throw new Error(...)` | **Not supported** | Use `Application.showMessage(ApplicationMessageType.Error, "...")` and `return` early. |
| `Promise`, `async`, `await` | **Not supported** | All scripts are synchronous. No async operations possible. |
| `setTimeout`, `setInterval` | **Not supported** | No deferred execution. |
| `document`, `window`, `navigator` | **Not supported** | No DOM or browser API access. |
| `fetch`, `XMLHttpRequest` | **Not supported** | No HTTP calls from scripts. |
| Dynamic typing (changing variable type) | **Not supported** | Once a variable is declared as `String`, it cannot hold a number. Types are fixed at declaration. |
| Implicit type coercion | **Not supported** | You cannot concatenate a number to a string without explicit conversion. Use `String(n)` or `n.toString()`. |
| `prototype` manipulation | **Not supported** | No class inheritance via prototype. |
| `new MyClass()` constructors | **Not supported** | Use `cast(Type.X, {...})` for SAC types. |
| Closures over mutable variables | **Unreliable** | Avoid relying on closure capture of loop variables. Use explicit index passing. |
| `arguments` object | **Not supported** | Use explicit parameters in function signatures. |
| Regular expressions (`/pattern/`) | **Not supported** | No regex. Use string methods: `indexOf`, `startsWith`, `endsWith`, `replace`. |
| `eval()` | **Not supported** | |
| Destructuring `const { a, b } = obj` | **Not supported** | Use explicit property access: `var a = obj.a;` |
| Spread operator `...` | **Not supported** | |
| Arrow functions `() => {}` | **Not supported** | Use `function(param) {}` syntax. |
| Template literals `` `Hello ${name}` `` | **Not supported** | Use string concatenation: `"Hello " + name` |
| `let` / `const` | **Not supported** | Use `var` only. |
| `for...of` | **Not supported** | Use classic `for (var i = 0; i < arr.length; i++)` |
| `forEach`, `map`, `filter`, `reduce` | **Not supported** | Use explicit `for` loops with index. |
| `JSON.parse`, `JSON.stringify` | **Not supported** | No JSON serialization. |
| `Math.random()` | **Unavailable** | No randomness. |
| `console.error`, `console.warn` | **Partially** | Only `console.log()` works (visible in browser dev tools). |

### Error Handling Pattern (without try/catch)

```javascript
// Pattern: validate → guard → act → show result

var idVal = InputField_ID.getValue().trim();

// Guard: validate before calling APIs
if (!idVal || idVal === "") {
    Application.showMessage(ApplicationMessageType.Error, "ID is required");
    return;   // ← early return is the only "error exit"
}

// Check Data Action result status
var result = DataAction_1.execute();
if (result.status !== DataActionExecutionResponseStatus.Success) {
    Application.showMessage(ApplicationMessageType.Error, "Data Action failed");
    return;
}

Application.showMessage(ApplicationMessageType.Success, "Done");
```

### Type System Rules

```javascript
// Correct: declare type explicitly
var memberName = "";          // String
var count = 0;                // Integer/number
var isActive = false;         // Boolean

// Wrong: do not reassign a different type
var x = 0;
x = "hello";   // ← RUNTIME ERROR in SAC scripts

// String conversion
var id = 42;
var idStr = id.toString();    // explicit conversion required
var label = "Member " + id.toString();
```

---

## Critical Limits — Read Before Writing Any Script

| # | Limit | Detail |
|---|---|---|
| L1 | **Generic dimensions only** | `createMembers`, `updateMembers`, `deleteMembers` work only on generic dimensions. Account, Version, Date, and Organization dimensions are not supported. |
| L2 | **CURRENCY property blocked on create** | Even on generic dimensions, the `CURRENCY` property cannot be set via `createMembers`. It can be set with `updateMembers` after creation. |
| L3 | **getMembers hard cap** | `getMembers()` returns at most 1000 members. For larger dimensions, use a filtered data source table's `getResultSet()` instead. |
| L4 | **No async** | SAC scripts run synchronously. There is no `await`, no Promise, no setTimeout. Execution happens sequentially on the click event. |
| L5 | **No DOM access** | No `document`, `window`, `fetch`, or browser APIs. Use only the SAC scripting namespace. |
| L6 | **Delete requires data cleared first** | A member with booked transactional data cannot be deleted directly. Run a Data Action to zero out / delete the data first, then call `deleteMembers`. |
| L7 | **Dropdown addItem vs setItems** | There is no `setItems()` bulk-load method. You must call `addItem(id, description)` per item inside a loop. Always call `removeAllItems()` first to avoid duplicates on re-populate. |
| L8 | **Popup cannot return a value** | A popup's OK button runs its own script. Values from popup widgets must be read by the OK button's script directly from the widget IDs (e.g. `InputField_Name.getValue()`). |
| L9 | **Table.getSelections() is dimension-specific** | `getSelections()` returns selected rows but requires dimension context. If the table has no row selected, it returns an empty array — always guard with a length check. |
| L10 | **refreshData is async** | `Application.refreshData([ds])` triggers a refresh but does not block. UI may show stale values for a moment after a create/update. If the form re-reads data immediately after, add the re-read inside the refresh callback if available, or use a PopUp to decouple. |
| L11 | **Script variables are page-scoped** | Script variables declared at page level are shared across all scripts on that page. Declare them once at page level; do not re-declare inside button scripts or you get shadowing bugs. |
| L12 | **Case-sensitive property keys** | Property names in `createMembers` / `updateMembers` must exactly match the technical property name in the model. `"description"` ≠ `"Description"`. |
| L13 | **Version/Account/Date/Org: read only via table** | `getMembers()` on Version, Account, Date, Organization dimensions is unsupported. Use a hidden table's `getDataSource().getResultSet()` to read these dimension members. |
| L14 | **No script-level error recovery** | There is no try/catch. A runtime error in a script stops that script entirely with a generic "Script Execution Failed" message. Prevent errors with pre-validation; never rely on catching them. |

---

## Core API Reference

### PlanningModel — Member Operations

```javascript
// READ all members of a dimension (max 1000)
var members = PlanningModel_1.getMembers("Product");
// members[i].id         → member technical ID
// members[i].description → display description
// members[i].properties  → object with property key/value pairs
// members[i].hierarchies → object with hierarchy parentId entries

// CREATE a new member
var newMember = cast(Type.PlanningModelMember, {
    id: "P001",
    description: "New Product",
    properties: {
        Manager: "John Smith",
        Region: "APAC",
        Status: "Active"
    },
    hierarchies: {
        H1: { parentId: "AllProducts" }
    }
});
PlanningModel_1.createMembers("Product", newMember);

// UPDATE an existing member's properties
var updMember = cast(Type.PlanningModelMember, {
    id: "P001",
    properties: {
        Manager: "Jane Doe",
        Status: "Inactive"
    }
});
PlanningModel_1.updateMembers("Product", updMember);

// DELETE a member
var delMember = cast(Type.PlanningModelMember, { id: "P001" });
PlanningModel_1.deleteMembers("Product", delMember);
```

> **Note**: `cast(Type.PlanningModelMember, {...})` is required — do not pass a plain object literal directly.

---

### Dropdown Widget

```javascript
// Populate from planning model members
Dropdown_Status.removeAllItems();
var members = PlanningModel_1.getMembers("StatusDim");
for (var i = 0; i < members.length; i++) {
    Dropdown_Status.addItem(members[i].id, members[i].description);
}

// Populate from a static list
Dropdown_Region.removeAllItems();
Dropdown_Region.addItem("APAC", "Asia Pacific");
Dropdown_Region.addItem("EMEA", "Europe, Middle East & Africa");
Dropdown_Region.addItem("AMER", "Americas");

// Read selected value
var selectedId = Dropdown_Region.getSelectedKey();
var selectedText = Dropdown_Region.getSelectedText();

// Set selected value (pre-fill for Edit form)
Dropdown_Region.setSelectedKey("APAC");

// Event: onSelect fires when user changes the selection
// (place logic in the Dropdown widget's onSelect event)
```

---

### InputField Widget

```javascript
// Read value
var nameVal = InputField_Name.getValue();

// Set value (pre-fill for Edit form)
InputField_Name.setValue("Existing Product Name");

// Clear field
InputField_Name.setValue("");

// Validation pattern
var nameVal = InputField_Name.getValue();
if (!nameVal || nameVal.trim() === "") {
    Application.showMessage(ApplicationMessageType.Error, "Name is required");
    return;
}
```

---

### Popup Widget

```javascript
// Open popup
Popup_AddForm.open();

// Close popup (call from the OK or Cancel button inside the popup)
Popup_AddForm.close();

// Pattern: OK button script reads from widgets inside the popup
// (The popup itself cannot "return" a value — read widgets directly)
var nameVal = InputField_Name.getValue();
var regionKey = Dropdown_Region.getSelectedKey();
```

---

### Table Widget — Read Selections

```javascript
// Get selected row(s)
var ds = Table_Products.getDataSource();
var selections = Table_Products.getSelections();

if (selections.length === 0) {
    Application.showMessage(ApplicationMessageType.Error, "Please select a row first");
    return;
}

// Read a dimension member from the selected row
var selectedMember = ds.getResultMember("Product", selections[0]);
var memberId = selectedMember.id;
var memberProps = selectedMember.properties;

// Pre-fill edit form from selection
InputField_Name.setValue(selectedMember.description);
Dropdown_Region.setSelectedKey(memberProps["Region"]);
```

---

### Application Utilities

```javascript
// Show message to user
Application.showMessage(ApplicationMessageType.Success, "Member created successfully");
Application.showMessage(ApplicationMessageType.Error, "ID is required");
Application.showMessage(ApplicationMessageType.Info, "Processing...");

// Refresh data sources after changes
Application.refreshData([Table_Products.getDataSource()]);

// Get current user
var userId = Application.getUserInfo().id;

// Console log (visible in browser dev tools)
console.log("Member ID: " + memberId);
```

---

### ArrayUtils — Deduplication Pattern

```javascript
// When building a dropdown from a property with duplicate values
var seen = ArrayUtils.create(Type.string);
var members = PlanningModel_1.getMembers("Product");
Dropdown_Category.removeAllItems();

for (var i = 0; i < members.length; i++) {
    var cat = members[i].properties["Category"];
    if (cat && ArrayUtils.indexOf(seen, cat) === -1) {
        ArrayUtils.push(seen, cat);
        Dropdown_Category.addItem(cat, cat);
    }
}
```

---

## Pre-Fill Patterns — Hardcoded and Dynamic Values

### Hardcoded default values

Use hardcoded defaults when the value is fixed by design (e.g. Status = "Active" on new member):

```javascript
// Pre-fill on Add button click
InputField_Year.setValue("2025");
Dropdown_Status.setSelectedKey("Active");
Dropdown_Region.setSelectedKey("APAC");
```

### Pre-fill from another dimension's members

When a dropdown should show members of a dimension that is NOT the one being edited:

```javascript
// E.g. Parent dropdown populated from the same dimension's hierarchy root
Dropdown_Parent.removeAllItems();
var allMembers = PlanningModel_1.getMembers("CostCenter");
for (var i = 0; i < allMembers.length; i++) {
    // Only show top-level members as candidate parents
    var m = allMembers[i];
    if (m.properties["IsParent"] === "Y") {
        Dropdown_Parent.addItem(m.id, m.description);
    }
}

// E.g. Dropdown populated from a DIFFERENT model dimension (e.g. Region)
Dropdown_Region.removeAllItems();
var regions = PlanningModel_1.getMembers("Region");
for (var i = 0; i < regions.length; i++) {
    Dropdown_Region.addItem(regions[i].id, regions[i].description);
}
```

### Pre-fill Version/Account members via hidden table (L13 workaround)

`getMembers()` does not work on Version, Account, Date, or Organization dimensions. Use a hidden table instead:

```javascript
// Layout: add a hidden table (setVisible(false)) bound to the model
// Configure it to show the Version dimension on rows
// Then read from it:

Dropdown_Version.removeAllItems();
var versionRows = Table_Hidden_Versions.getDataSource().getResultSet();
for (var i = 0; i < versionRows.length; i++) {
    var row = versionRows[i];
    // Row columns depend on table configuration — use the dimension member key
    Dropdown_Version.addItem(row["Version"], row["Version"]);
}
```

### Pre-fill from current user context

```javascript
// Get logged-in user ID
var userId = Application.getUserInfo().id;

// Look up a property of that user from a User dimension
var userMembers = PlanningModel_1.getMembers("User");
var userRegion = "";
for (var i = 0; i < userMembers.length; i++) {
    if (userMembers[i].properties["SystemID"] === userId) {
        userRegion = userMembers[i].properties["Region"];
        break;
    }
}

// Pre-fill form with user's default region
if (userRegion !== "") {
    Dropdown_Region.setSelectedKey(userRegion);
}
```

### Pre-fill Edit form from table selection

```javascript
// Full pre-fill pattern for Edit mode
var selections = Table_Members.getSelections();
if (selections.length === 0) {
    Application.showMessage(ApplicationMessageType.Error, "Select a row to edit");
    return;
}
var ds = Table_Members.getDataSource();
var sel = ds.getResultMember("CostCenter", selections[0]);

// Text fields
InputField_ID.setValue(sel.id);
InputField_Description.setValue(sel.description);

// Properties → dropdowns / inputs
Dropdown_Region.setSelectedKey(sel.properties["Region"]);
Dropdown_Status.setSelectedKey(sel.properties["Status"]);
InputField_Manager.setValue(sel.properties["Manager"]);

// Hierarchy parent
var parentId = "";
if (sel.hierarchies && sel.hierarchies["H1"] && sel.hierarchies["H1"].parentId) {
    parentId = sel.hierarchies["H1"].parentId;
}
Dropdown_Parent.setSelectedKey(parentId);
```

---

## Cross-Dimension Lookup Patterns

When a form field in dimension A must show values from dimension B:

### Pattern 1 — Static cross-dimension list

If dimension B is small and stable, hardcode its values:

```javascript
Dropdown_ProductLine.removeAllItems();
Dropdown_ProductLine.addItem("L01", "Automotive");
Dropdown_ProductLine.addItem("L02", "Industrial");
Dropdown_ProductLine.addItem("L03", "Consumer");
```

### Pattern 2 — Read dimension B members from model

```javascript
// onInitialization: populate a dropdown from a related dimension
Dropdown_BusinessUnit.removeAllItems();
var buMembers = PlanningModel_1.getMembers("BusinessUnit");
for (var i = 0; i < buMembers.length; i++) {
    Dropdown_BusinessUnit.addItem(buMembers[i].id, buMembers[i].description);
}
```

### Pattern 3 — Cascading dropdown (selection in A filters B)

SAC has no native cascading dropdown. Simulate it by repopulating dropdown B in dropdown A's `onSelect` event:

```javascript
// Dropdown_Region onSelect event
var selectedRegion = Dropdown_Region.getSelectedKey();

Dropdown_CostCenter.removeAllItems();
var allCC = PlanningModel_1.getMembers("CostCenter");
for (var i = 0; i < allCC.length; i++) {
    // Only show cost centers belonging to selected region
    if (allCC[i].properties["Region"] === selectedRegion) {
        Dropdown_CostCenter.addItem(allCC[i].id, allCC[i].description);
    }
}
```

> **Limitation**: If there are >1000 members, filtering inside `getMembers()` is not possible — you get the first 1000 unfiltered. Use a table-backed approach (L13) for large dimensions.

### Pattern 4 — Read a property from a related member

When you need to look up a value from another dimension based on selection:

```javascript
// User selected a Cost Center — look up its Manager from a different property
var selectedCC = Dropdown_CostCenter.getSelectedKey();
var allCC = PlanningModel_1.getMembers("CostCenter");
var managerName = "";

for (var i = 0; i < allCC.length; i++) {
    if (allCC[i].id === selectedCC) {
        managerName = allCC[i].properties["Manager"];
        break;   // ← always break once found; do not continue looping
    }
}

InputField_Manager.setValue(managerName);
```

---

## /script-form — Full CRUD Form Pattern

### Layout

```
[Page]
  Table_Members          ← Shows all members; user clicks to select
  Button_Add             ← Opens Popup_Form in Add mode
  Button_Edit            ← Opens Popup_Form in Edit mode, pre-filled
  Button_Delete          ← Deletes selected member (with confirmation)

  Popup_Form
    InputField_ID          ← Member technical ID (disabled in Edit mode)
    InputField_Description ← Display description
    Dropdown_Parent        ← Parent in hierarchy H1
    Dropdown_Status        ← From static list or dimension
    [any additional property fields]
    Button_OK              ← Commits create or update
    Button_Cancel          ← Closes popup without saving
```

---

### Script Object — Page Level Variables

Declare at page level (not inside button scripts):

```javascript
// Script variable: current form mode
// Type: String
var g_form_mode = "";        // "ADD" or "EDIT"
var g_edit_member_id = "";   // ID of member being edited
```

---

### onInitialization (Page)

```javascript
// Populate parent dropdown from model
Dropdown_Parent.removeAllItems();
var parents = PlanningModel_1.getMembers("Product");
for (var i = 0; i < parents.length; i++) {
    Dropdown_Parent.addItem(parents[i].id, parents[i].description);
}

// Populate status dropdown (static)
Dropdown_Status.removeAllItems();
Dropdown_Status.addItem("Active", "Active");
Dropdown_Status.addItem("Inactive", "Inactive");
```

---

### Button_Add — onClick

```javascript
// Set mode flag
g_form_mode = "ADD";
g_edit_member_id = "";

// Clear all form fields
InputField_ID.setValue("");
InputField_Description.setValue("");
Dropdown_Parent.setSelectedKey("");
Dropdown_Status.setSelectedKey("Active");   // sensible default

// Enable the ID field (locked in Edit mode)
InputField_ID.setEnabled(true);

// Open form
Popup_Form.open();
```

---

### Button_Edit — onClick

```javascript
// Guard: must have a selection
var selections = Table_Members.getSelections();
if (selections.length === 0) {
    Application.showMessage(ApplicationMessageType.Error, "Select a member to edit");
    return;
}

// Set mode flag
g_form_mode = "EDIT";
var ds = Table_Members.getDataSource();
var sel = ds.getResultMember("Product", selections[0]);
g_edit_member_id = sel.id;

// Pre-fill form
InputField_ID.setValue(sel.id);
InputField_ID.setEnabled(false);           // ID is immutable on edit
InputField_Description.setValue(sel.description);
Dropdown_Status.setSelectedKey(sel.properties["Status"]);

// Set parent from hierarchy
var parentId = "";
if (sel.hierarchies && sel.hierarchies["H1"]) {
    parentId = sel.hierarchies["H1"].parentId;
}
Dropdown_Parent.setSelectedKey(parentId);

// Open form
Popup_Form.open();
```

---

### Button_Delete — onClick

```javascript
// Guard: must have a selection
var selections = Table_Members.getSelections();
if (selections.length === 0) {
    Application.showMessage(ApplicationMessageType.Error, "Select a member to delete");
    return;
}

var ds = Table_Members.getDataSource();
var sel = ds.getResultMember("Product", selections[0]);
var memberId = sel.id;

// ⚠ Run a Data Action to clear transactional data first if any exists
// Then delete the member
var delMember = cast(Type.PlanningModelMember, { id: memberId });
PlanningModel_1.deleteMembers("Product", delMember);

Application.refreshData([Table_Members.getDataSource()]);
Application.showMessage(ApplicationMessageType.Success, "Member deleted: " + memberId);
```

---

### Button_OK (inside Popup_Form) — onClick

```javascript
// Read form values
var idVal = InputField_ID.getValue().trim();
var descVal = InputField_Description.getValue().trim();
var parentVal = Dropdown_Parent.getSelectedKey();
var statusVal = Dropdown_Status.getSelectedKey();

// Validation
if (!idVal) {
    Application.showMessage(ApplicationMessageType.Error, "ID is required");
    return;
}
if (!descVal) {
    Application.showMessage(ApplicationMessageType.Error, "Description is required");
    return;
}
if (!parentVal) {
    Application.showMessage(ApplicationMessageType.Error, "Parent is required");
    return;
}

if (g_form_mode === "ADD") {
    // CREATE
    var newMember = cast(Type.PlanningModelMember, {
        id: idVal,
        description: descVal,
        properties: {
            Status: statusVal
        },
        hierarchies: {
            H1: { parentId: parentVal }
        }
    });
    PlanningModel_1.createMembers("Product", newMember);
    Application.showMessage(ApplicationMessageType.Success, "Member created: " + idVal);

} else if (g_form_mode === "EDIT") {
    // UPDATE — id is g_edit_member_id (not idVal, which is disabled)
    var updMember = cast(Type.PlanningModelMember, {
        id: g_edit_member_id,
        description: descVal,
        properties: {
            Status: statusVal
        },
        hierarchies: {
            H1: { parentId: parentVal }
        }
    });
    PlanningModel_1.updateMembers("Product", updMember);
    Application.showMessage(ApplicationMessageType.Success, "Member updated: " + g_edit_member_id);
}

// Refresh table and close popup
Application.refreshData([Table_Members.getDataSource()]);
Popup_Form.close();
```

---

### Button_Cancel (inside Popup_Form) — onClick

```javascript
Popup_Form.close();
```

---

## /script-debug — Diagnostic Checklist

**Dropdown shows nothing after populate**
1. Did you call `removeAllItems()` first? Without it, `addItem` appends to stale items.
2. Is `getMembers()` returning results? Log `members.length` to console.
3. Is the dimension name exact? `"product"` ≠ `"Product"`.
4. Is the populate code in `onInitialization`? It must run at app start.

**createMembers runs but member doesn't appear**
1. Is the dimension a **generic** dimension? (L1 — account/date/version/org not supported)
2. Did you use `cast(Type.PlanningModelMember, {...})`? Plain object literals fail silently.
3. Is the hierarchy name correct? Wrong hierarchy name means the member is created but orphaned.
4. Did `refreshData` run after the create? Without it the table doesn't update.
5. Is the `id` unique? Duplicate IDs fail silently in some versions.

**updateMembers does nothing**
1. Property key case — `"Status"` ≠ `"status"`. Check the technical property name in the model.
2. The `id` field must be an existing member. Check for typos.
3. CURRENCY property cannot be updated this way (L2 — SAP known bug workaround: use a separate admin step).

**deleteMembers fails**
1. Member has booked transactional data — run a Data Action to clear it first (L6).
2. Member is referenced by another dimension (e.g. a parent) — remove references first.
3. Check that the member ID is the technical ID, not the description.

**Table selection returns empty**
1. Is the table configured to allow row selection? (Table widget → Display settings → Allow selection)
2. `getSelections()` returns selected rows — user must click a row first.
3. Dimension name in `getResultMember("Product", sel)` must match the table's row dimension.

**Form mode bug (edit overwrites wrong member)**
1. `g_form_mode` and `g_edit_member_id` must be page-level script variables, not declared inside button scripts.
2. In Edit mode, the OK script must use `g_edit_member_id` for the update, not `InputField_ID.getValue()` (which is disabled and may be empty).

---

## /script-widget — Widget Quick Reference

### Dropdown

| Method | Description |
|---|---|
| `removeAllItems()` | Clear all options — always call before repopulating |
| `addItem(id, description)` | Add one option |
| `getSelectedKey()` | Returns the selected item's ID |
| `getSelectedText()` | Returns the selected item's display text |
| `setSelectedKey(id)` | Programmatically set selection |
| `setEnabled(bool)` | Enable/disable widget |
| `setVisible(bool)` | Show/hide widget |

Events: `onSelect` — fires when user changes selection.

### InputField

| Method | Description |
|---|---|
| `getValue()` | Returns current text value (always a string) |
| `setValue(text)` | Sets the field value |
| `setEnabled(bool)` | Enable/disable — disabled fields can still be read |
| `setVisible(bool)` | Show/hide |

No change event — reads happen on button click only.

### Popup

| Method | Description |
|---|---|
| `open()` | Show the popup |
| `close()` | Close the popup |

No return value. Popups cannot "return" data — read widget values from inside the popup's button scripts directly.

### Table

| Method | Description |
|---|---|
| `getDataSource()` | Returns the data source object |
| `getSelections()` | Returns array of selected row references |
| `setVisible(bool)` | Show/hide |

### DataSource (from getDataSource())

| Method | Description |
|---|---|
| `getResultSet()` | Returns all rows in the current table result |
| `getResultMember(dimension, selection)` | Returns a member object from a row selection |
| `getMembers(dimension)` | Returns dimension members (max 1000) |
| `setDimensionFilter(dimension, filter)` | Apply a filter — format: `"[Dim].[H1].&[ID]"` |
| `removeFilter(dimension)` | Clear a filter |

---

## Pre-Build Checklist for Any Form

- [ ] Dimension is **generic type** (not Account, Version, Date, or Organization)
- [ ] All property technical names confirmed in the model (case-sensitive)
- [ ] Hierarchy name confirmed (e.g. `H1`, `H2`)
- [ ] Page-level script variables declared for form mode and edit member ID
- [ ] `onInitialization` populates all dropdowns at startup
- [ ] `removeAllItems()` called before every dropdown population loop
- [ ] `cast(Type.PlanningModelMember, {...})` used for all member operations
- [ ] Delete button runs Data Action first if dimension has transactional data
- [ ] `Application.refreshData([...])` called after every create/update/delete
- [ ] All inputs validated before calling the planning model API
- [ ] Error messages use `ApplicationMessageType.Error` (not raw strings)

---

## Known SAC Scripting Constraints Summary

| Constraint | Workaround |
|---|---|
| No `setItems()` bulk method on Dropdown | Loop with `addItem()` + `removeAllItems()` first |
| No async/await | All logic must be synchronous on the button click |
| No DOM access | Use only SAC scripting APIs |
| CURRENCY property blocked on createMembers | Create without it; update in a second call |
| deleteMembers fails with booked data | Data Action to clear data → then delete |
| getMembers capped at 1000 | Use table `getResultSet()` for larger dims |
| Popup cannot return a value | Read widget values directly inside OK button script |
| Script variables re-declared inside scripts shadow page variables | Declare once at page level only |
| refreshData non-blocking | Decouple with Popup or accept momentary stale display |
| Properties are case-sensitive | Use exact technical name from model property list |

---

Maintainer: Cognivion SAC Practice | SAC 2026.15 | github.com/anilkumarkn100/sac-da-knowledge
