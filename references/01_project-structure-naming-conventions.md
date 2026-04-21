# Skill 01 — Project Structure & Naming Conventions

**Purpose:** Enforce a consistent folder structure and naming convention across all Mendix modules to ensure clarity, discoverability, and team-wide alignment.

---

## Folder Structure

```
<ModuleName>/
├── microflows/
│   ├── actions/          # ACT_ flows triggered by user or system events
│   ├── sub/              # SUB_ reusable logic called from other microflows
│   ├── validations/      # VAL_ input/data validation flows
│   └── integrations/     # INT_ external API / service calls
├── nanoflows/            # NF_ all nanoflow logic (UI/offline)
├── java_actions/         # JA_ custom Java actions
├── javascript_actions/   # JSA_ custom JavaScript actions
├── widgets/              # Custom pluggable widgets (React)
├── styles/
│   ├── web/              # .scss files for web styling
│   └── native/           # .js style files for native
├── domain_models/        # Entity groupings per bounded context
├── pages/
│   ├── forms/
│   ├── dashboards/
│   └── admin/
├── snippets/             # Reusable page snippets
├── constants/            # App-wide constant values
└── enumerations/         # Enum definitions
```

---

## Naming Conventions

### Microflows

| Prefix | Usage | Example |
|--------|-------|---------|
| `ACT_` | Action triggered by a button, event, or timer | `ACT_CreateEmployee` |
| `SUB_` | Sub-microflow called by another microflow | `SUB_ValidateEmail` |
| `VAL_` | Validation logic only, returns Boolean | `VAL_IsEmployeeEligible` |
| `INT_` | External integration calls | `INT_OracleFusion_GetEmployee` |

**Full format:** `{PREFIX}_{Entity/Context}_{Action}`
**Examples:**
- `ACT_Employee_Create`
- `SUB_Address_Format`
- `VAL_LeaveRequest_CheckBalance`
- `INT_OracleFusion_SyncEmployee`

---

### Nanoflows

| Prefix | Usage | Example |
|--------|-------|---------|
| `NF_` | All nanoflows regardless of purpose | `NF_Employee_ShowDetails` |

**Full format:** `NF_{Context}_{Action}`

---

### Java Actions

| Prefix | Usage | Example |
|--------|-------|---------|
| `JA_` | All Java actions | `JA_Crypto_HashPassword` |

**Full format:** `JA_{Domain}_{Action}`

---

### JavaScript Actions

| Prefix | Usage | Example |
|--------|-------|---------|
| `JSA_` | All JavaScript actions | `JSA_Browser_GetCurrentURL` |

**Full format:** `JSA_{Context}_{Action}`

---

### Pages

| Suffix | Usage | Example |
|--------|-------|---------|
| `_Overview` | List/grid page | `Employee_Overview` |
| `_Detail` | Detail/form page | `Employee_Detail` |
| `_NewEdit` | Combined new/edit popup | `LeaveRequest_NewEdit` |
| `_Dashboard` | Dashboard page | `HR_Dashboard` |

---

### Entities & Attributes

- **Entities:** PascalCase — `EmployeeProfile`, `LeaveRequest`
- **Attributes:** camelCase — `employeeId`, `startDate`, `isActive`
- **Associations:** `{Parent}_{Child}` — `Employee_LeaveRequest`
- **Enumerations:** PascalCase with prefix — `ELeaveType`, `EEmployeeStatus`

---

### Constants

Format: `CONST_{MODULE}_{NAME}`
Example: `CONST_HR_MaxLeaveDays`, `CONST_API_BaseURL`

---

## Rules for AI Code Generation

- Always place generated microflows in the matching subfolder under `microflows/`
- Always apply the correct prefix based on the microflow type
- Never use generic names like `Microflow1` or `NewMicroflow`
- Always include the entity or domain context in the name
- Pages must always include the layout suffix (`_Overview`, `_Detail`, etc.)

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Use prefixes consistently on all flows | Name flows without prefix or context |
| Keep module folders scoped to bounded contexts | Mix integration logic with action logic |
| Apply PascalCase to entities | Use abbreviations without a glossary |
| Keep widget folder clean (1 folder per widget) | Place widget code outside `widgets/` |
| Document each constant with a description | Use magic numbers/strings in flows |

---

## Example Usage

> **Scenario:** Building a Leave Request approval feature.

```
ACT_LeaveRequest_Submit         → triggers validation + save
VAL_LeaveRequest_CheckBalance   → validates available days
SUB_Notification_SendApproval   → sends notification email
INT_OracleFusion_PostLeave      → syncs approved leave to Oracle
NF_LeaveRequest_ShowConfirm     → nanoflow: show confirmation dialog
```
