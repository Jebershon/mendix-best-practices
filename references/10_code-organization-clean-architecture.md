# Skill 10 — Code Organization & Clean Architecture

**Purpose:** Define patterns for separation of concerns, modular design, reusability, and avoiding duplication in Mendix applications.

---

## Core Architectural Principles

1. **Separation of Concerns** — each microflow/nanoflow/widget has exactly one job
2. **Single Responsibility** — if you need "and" to describe a flow, split it
3. **DRY (Don't Repeat Yourself)** — duplicated logic becomes a SUB_ microflow
4. **Modular Boundaries** — each module exposes a clean public interface
5. **Dependency direction** — lower modules (Shared, Logging) never depend on higher modules (HR, Finance)

---

## Marketplace Module Reuse (Check Before You Build)

Before writing any new microflow, sub-microflow, or Java action, **always check whether the functionality already exists in a downloaded Marketplace module.** All required Marketplace modules will be pre-installed at the start of the project — your job is to know what they provide and reuse them.

### Why This Matters

Marketplace modules like **Community Commons**, **Encryption**, **Excel Importer**, and others ship with battle-tested, production-ready microflows and Java actions. Reimplementing these wastes time and introduces bugs that have already been solved upstream.

### Mandatory Lookup Order

Before building new logic, check in this order:

```
1. Marketplace modules (downloaded and available in the project)
2. Shared_Module / Common custom SUB_ flows already built in this project
3. Build new only if neither above covers the requirement
```

### Key Marketplace Modules & What to Reuse

#### Community Commons (`CommunityCommons`)
The most commonly used module. Always check here first.

| Functionality | Reuse from Community Commons |
|---------------|------------------------------|
| String formatting | `CommunityCommons.StringLeftPad`, `FormatString` |
| Null/empty checks | `CommunityCommons.IsNullOrEmpty` |
| Date arithmetic | `CommunityCommons.DateAdd`, `DateDiff` |
| UUID generation | `CommunityCommons.CreateUUID` |
| Deep clone object | `CommunityCommons.Clone` |
| List operations | `CommunityCommons.ListContains`, `Intersect`, `Subtract` |
| File utilities | `CommunityCommons.GetFileSize`, `CopyFileDocument` |
| XPath first result | `CommunityCommons.GetFirstItemFromList` |

> ✅ **Rule:** If Community Commons has it, use it. Do not rebuild string, date, list, or UUID utilities from scratch.

#### Encryption Module (`Encryption`)

| Functionality | Reuse |
|---------------|-------|
| Encrypt string | `Encryption.EncryptString` |
| Decrypt string | `Encryption.DecryptString` |
| Hash value | `Encryption.MD5Hash` |

> ✅ **Rule:** Never write custom Java actions for encryption if the Encryption module is installed.

#### Excel Importer / Exporter

| Functionality | Reuse |
|---------------|-------|
| Import Excel file | `ExcelImporter` module flows |
| Export to Excel | `MxModelReflection` + `ExcelExporter` flows |

#### Email Module (`EmailTemplate` / `MendixSSO` etc.)

| Functionality | Reuse |
|---------------|-------|
| Send templated email | `EmailTemplate.SendEmail` |
| Queue email | `EmailTemplate.QueueEmail` |

#### Other Common Modules

| Module | What it provides |
|--------|-----------------|
| `MxModelReflection` | Runtime entity/attribute inspection |
| `SAML` / `MendixSSO` | Authentication flows — never rebuild |
| `Rest Services` | Reusable REST patterns |
| `UnitTesting` | Test microflow framework |

---

### How to Check (Workflow for Every New Flow)

```
STEP 1: Identify what you need
  → "I need to generate a UUID for a correlation ID"

STEP 2: Search in Marketplace modules
  → Open Studio Pro > App Explorer > CommunityCommons
  → Search: "UUID"
  → Found: CommunityCommons.CreateUUID ✅

STEP 3: Call it from your microflow
  → ACT_Log_GenerateCID()
       → CommunityCommons.CreateUUID() → $CID
       → Return $CID

STEP 4: Document the dependency
  → Add comment in microflow: "Uses CommunityCommons.CreateUUID"
```

If the Marketplace module does NOT cover the requirement, then build a new `SUB_` or `JA_` — and document that no existing utility was found.

---

### Marketplace Reuse Documentation Rule

When you reuse a Marketplace flow or Java action, add an annotation in the calling microflow:

```
Annotation: "Delegates to CommunityCommons.FormatString — do not replace with custom logic."
```

This prevents future developers from unknowingly duplicating it.

---

## Module Layer Architecture

Structure your Mendix app in layers. Dependencies only flow **downward**:

```
┌─────────────────────────────────────────────────┐
│         Application Modules (Feature Layer)      │
│   HR_Module  │  Finance_Module  │  Portal_Module  │
└─────────────────────────────────────────────────┘
                        │ depends on
┌─────────────────────────────────────────────────┐
│           Shared / Common Module                 │
│   SUB_Notification  │  SUB_Validation  │  Utils   │
└─────────────────────────────────────────────────┘
                        │ depends on
┌─────────────────────────────────────────────────┐
│          Foundation Modules                      │
│      Logging_Module  │  Security_Module          │
└─────────────────────────────────────────────────┘
                        │ depends on
┌─────────────────────────────────────────────────┐
│            System / Mendix Runtime               │
└─────────────────────────────────────────────────┘
```

**Rule:** `HR_Module` may call `Shared_Module`. `Shared_Module` may call `Logging_Module`. `Logging_Module` must never call `HR_Module`.

---

## Separation of Concerns — Microflow Layers

Each ACT_ microflow should delegate to focused sub-flows:

```
ACT_LeaveRequest_Submit($Request)
  │
  ├── [Layer 1: Input Validation]
  │     → VAL_LeaveRequest_Validate($Request)
  │
  ├── [Layer 2: Business Logic]
  │     → SUB_LeaveRequest_CalculateDays($Request)
  │     → SUB_LeaveBalance_Deduct($Request)
  │
  ├── [Layer 3: Persistence]
  │     → Commit $Request (within error handler)
  │
  ├── [Layer 4: Integration]
  │     → INT_OracleFusion_PostLeave($Request) (async if possible)
  │
  └── [Layer 5: Notification]
        → SUB_Notification_SendApproval($Request)
```

Each layer can be maintained, replaced, or tested independently.

---

## Reusability Patterns

### Pattern 1: Common Validation Sub-Flow

Instead of repeating email validation in 10 microflows:

```
SUB_Validate_Email($Email) → Boolean
  → Use regex or JSA_Validate_EmailFormat
  → Return true/false

// Called from:
VAL_Employee_Validate($Employee)
VAL_Contact_Validate($Contact)
VAL_Registration_Validate($Registration)
```

### Pattern 2: Notification Dispatcher

Single notification sub-flow handles all channels:

```
SUB_Notification_Send($Template, $Recipient, $Subject, $Body)
  → If notificationType = Email:  Call email action
  → If notificationType = Push:   Call push notification action
  → If notificationType = InApp:  Create notification entity
  → Log: SUB_Log_Write(INFO, 'SUB_Notification_Send', 'Sent ' + $Template + ' to ' + $Recipient)
```

### Pattern 3: Audit Stamper

Set audit fields consistently via one sub-flow:

```
SUB_Audit_SetTimestamps($Object)
  → Change: $Object/LastModifiedDate = [%CurrentDateTime%]
  → Change: $Object/LastModifiedBy = $currentUser/Name
```

### Pattern 4: Generic Error Handler Sub-Flow

```
SUB_Error_Handle($FlowName, $CorrelationID)
  → SUB_Log_Write(ERROR, $FlowName, $latestError/message, $CorrelationID, $latestError/stackTrace)
  → Rollback
  → Return false
```

---

## Avoiding Duplication

**Before creating any new microflow, ask (in order):**
1. Does a **Marketplace module** already provide this? (CommunityCommons, Encryption, etc.)
2. Does this logic already exist in a **SUB_ microflow** in this project?
3. Can it be made generic with a parameter?
4. Is it a candidate for the Shared module?

**Signs of duplication:**
- Same XPath retrieve used in 5 different flows → create `SUB_Employee_GetByID($ID)`
- Same commit + log pattern in 10 ACT_ flows → extract to `SUB_Persist_Commit($Object, $FlowName)`
- Same notification trigger in 3 flows → extract to `SUB_Notification_SendApproval($Request)`

---

## Module Interface Rules

Each module should expose a **public interface** — a small set of ACT_ and SUB_ flows that other modules may call:

```
HR_Module Public Interface:
  ✅ ACT_Employee_Create
  ✅ ACT_Employee_Update
  ✅ SUB_Employee_GetByID
  ✅ SUB_Employee_IsActive

  ❌ SUB_Employee_InternalValidate  (private — not for cross-module use)
  ❌ SUB_HR_AuditInternal           (private)
```

**Rule:** Mark private sub-flows with `[PRIVATE]` in their description. Do not call them from outside their module.

---

## Code Organization Within Modules

Keep flows grouped logically within each module folder:

```
HR_Module/
├── microflows/
│   ├── actions/
│   │   ├── ACT_Employee_Create
│   │   ├── ACT_Employee_Update
│   │   └── ACT_LeaveRequest_Submit
│   ├── sub/
│   │   ├── SUB_Employee_GetByID
│   │   ├── SUB_Employee_FormatName
│   │   └── SUB_LeaveBalance_Check
│   ├── validations/
│   │   ├── VAL_Employee_Validate
│   │   └── VAL_LeaveRequest_Validate
│   └── integrations/
│       ├── INT_OracleFusion_GetEmployee
│       └── INT_OracleFusion_PostLeave
```

---

## Clean Microflow Design Rules

- **Max 15 activities** per microflow — split beyond that
- **No more than 2 levels of nesting** in decision trees — use sub-flows instead
- **Every decision must have both TRUE and FALSE paths** handled
- **Every loop must have a clear exit condition**
- **Annotate complex sections** with Mendix annotation blocks

---

## Constants Over Magic Values

```
// ❌ Wrong: Magic string
if $Request/Status = 'APR' then ...

// ✅ Right: Named constant
// CONST_HR_StatusApproved = 'Approved'
if $Request/Status = CONST_HR_StatusApproved then ...
```

---

## Dependency Management

- Never create **circular dependencies** between modules
- Track cross-module calls in a dependency map document
- Prefer **events/callbacks** (Mendix After Commit handlers) over direct cross-module calls where appropriate

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Check Marketplace modules before building new logic | Rebuild what Community Commons already provides |
| Annotate microflows that delegate to Marketplace flows | Use Marketplace flows silently with no documentation |
| One responsibility per microflow | Build multi-purpose "god" microflows |
| Extract repeated logic into SUB_ flows | Copy-paste the same 5 activities |
| Group related flows in descriptive subfolders | Put everything in one flat folder |
| Use constants for all configurable values | Hardcode values inside microflows |
| Document public module interfaces | Let every sub-flow be callable anywhere |
| Keep module dependencies one-directional | Create circular module dependencies |

---

## Example Usage

> **Scenario:** Refactoring a monolithic ACT_Employee_Onboard into clean architecture

**Before (monolithic):**
```
ACT_Employee_Onboard — 45 activities:
  Create user account + validate + set roles + create profile +
  send welcome email + sync to Oracle + create default tasks + log...
```

**After (clean, modular):**
```
ACT_Employee_Onboard($Employee) — 7 activities:
  ├── VAL_Employee_Validate($Employee)
  ├── SUB_Account_CreateUser($Employee)      → HR_Module/sub
  ├── SUB_Role_AssignDefault($Employee)      → HR_Module/sub
  ├── INT_OracleFusion_CreateEmployee($Employee) → HR_Module/integrations
  ├── SUB_Task_CreateOnboardingSet($Employee) → Task_Module (cross-module)
  ├── SUB_Notification_SendWelcome($Employee) → Shared_Module
  └── SUB_Log_Write(INFO, 'ACT_Employee_Onboard', 'Onboarding complete: ' + $Employee/Name)
```
