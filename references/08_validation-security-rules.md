# Skill 08 — Validation & Security Rules

**Purpose:** Enforce entity access control, input validation, role-based security, and secure API usage across all Mendix applications.

---

## Core Security Principles

1. **Security is not optional** — every entity, microflow, and API call must have access control defined
2. **Default deny** — start with no access and grant explicitly
3. **Validate on the server** — client-side validation is UX only; server-side validation is mandatory
4. **Never expose secrets** — API keys, credentials, tokens must live in constants or encrypted attributes
5. **Least privilege** — each role gets only what it needs, nothing more

---

## Entity Access Rules

### Domain Model Security Checklist

For every entity:

- [ ] Access rules defined for every module role
- [ ] Default access = **No access** for all roles unless explicitly granted
- [ ] Read-only attributes set for audit fields (`CreatedDate`, `CreatedBy`, `ChangedDate`)
- [ ] Sensitive attributes (passwords, tokens) — **never readable** from pages
- [ ] XPath-based constraints for row-level security (e.g., user can only see own records)

### Access Rule Template

```
Entity: LeaveRequest
Rules:
  HR_Manager:
    - Create: ✅
    - Read:   ✅ (all attributes)
    - Update: ✅
    - Delete: ❌
  Employee:
    - Create: ✅ (only own records)
    - Read:   ✅ (XPath: [Employee = '[%CurrentUser%]'])
    - Update: ✅ (only if Status = 'Draft')
    - Delete: ❌
  Admin:
    - All:    ✅
```

### XPath Row-Level Security

```
// Employee can only see their own leave requests
[Employee_LeaveRequest/HR.Employee/SystemUser = '[%CurrentUser%]']

// User can only update records in Draft status
[Status = 'Draft' and Employee_LeaveRequest/HR.Employee/SystemUser = '[%CurrentUser%]']
```

---

## Input Validation Patterns

### Validation Microflow Template (VAL_)

```
VAL_LeaveRequest_Validate($Request)
  │
  ├── If $Request/StartDate = empty   → $Errors = $Errors + "Start date is required. "
  ├── If $Request/EndDate = empty     → $Errors = $Errors + "End date is required. "
  ├── If $Request/StartDate > $Request/EndDate → $Errors = $Errors + "Start date must be before end date. "
  ├── If $Request/Days <= 0           → $Errors = $Errors + "Days must be greater than zero. "
  ├── If length($Request/Reason) < 10 → $Errors = $Errors + "Reason must be at least 10 characters. "
  │
  ├── If $Errors != ''  → Show validation message($Errors) → Return false
  └── Return true
```

### Standard Validation Rules

| Field Type | Rule | Mendix Expression |
|------------|------|-------------------|
| Required string | Must not be empty | `$Obj/Field != ''` |
| Required object | Must not be empty | `$Obj/Association != empty` |
| Date range | Start before end | `$Obj/StartDate < $Obj/EndDate` |
| Positive number | Greater than zero | `$Obj/Amount > 0` |
| Email format | Valid email pattern | Use `SUB_Validate_Email($email)` with regex via JSA |
| String length | Min/max length | `length($Obj/Field) >= 5` |
| Enum not default | Must be selected | `$Obj/Status != EStatus.None` |

---

## Role-Based Security

### Module Roles (Recommended Standard)

Define roles per module with clear naming:

| Role Name | Description |
|-----------|-------------|
| `Admin` | Full access — system admin only |
| `Manager` | Approve/reject workflows, view reports |
| `Employee` | Self-service — own data only |
| `Viewer` | Read-only access |
| `Integration` | API/system account — no UI access |

### Microflow Access Rules

- **Every ACT_ microflow** must have module role access configured
- **SUB_ microflows** inherit access via calling flow — restrict access
- **INT_ microflows** should only be callable by the `Integration` role or `Admin`
- **Scheduled events** run as `System` — no additional role needed

### Page Access Rules

- Each page must have explicit allowed roles set
- Never leave pages with "All roles" unless it is a public-facing page
- Admin pages must only be accessible to `Admin` role

---

## Preventing Unauthorized Access

### Secure Microflow Pattern

```
ACT_LeaveRequest_Approve($RequestID, $CurrentUser)
  │
  ├── Retrieve $Request by $RequestID
  │
  ├── Security Check: Is $CurrentUser a Manager?
  │     → Retrieve $UserRole for $CurrentUser
  │     → If role != 'HR_Manager' and role != 'Admin':
  │           SUB_Log_Write(WARNING, "ACT_...", "Unauthorized access attempt by: " + $CurrentUser/Name)
  │           Return false / Show "Access denied"
  │
  ├── Business Logic: Update $Request/Status = 'Approved'
  └── Commit + Notify
```

### Critical Anti-Patterns to Avoid

| ❌ Anti-Pattern | ✅ Secure Alternative |
|----------------|----------------------|
| Pass entity ID in URL without server check | Retrieve server-side and verify ownership |
| Store passwords in plain text | Use `JA_Crypto_HashSHA256` |
| Expose API keys in JS actions | Store in encrypted constants, pass as param |
| Allow all roles on ACT_ microflows | Define role per microflow in security settings |
| Skip XPath constraints | Always apply XPath for user-scoped data |

---

## Secure API Usage

### REST/SOAP Call Security Rules

1. **Store API credentials** in Mendix Constants (encrypted where supported)
2. **Use HTTPS only** — never allow HTTP endpoints
3. **Validate response status** before processing
4. **Mask sensitive data in logs** — never log passwords, tokens, or PII
5. **Implement retry with backoff** for transient errors
6. **Set request timeouts** — never leave calls with no timeout

```
INT_OracleFusion_GetEmployee($EmployeeID)
  │
  ├── Set Headers: Authorization = "Bearer " + CONST_Oracle_Token (encrypted)
  ├── Set Timeout: 30 seconds
  ├── Call REST: GET /employees/{$EmployeeID}
  │
  ├── If HTTPStatus = 200: Parse response → return $Employee
  ├── If HTTPStatus = 401: SUB_Log_Write(ERROR, ..., "Auth failed") → return empty
  ├── If HTTPStatus = 404: Return empty (not found)
  └── If HTTPStatus >= 500: Throw error → handled by calling microflow's catch block
```

---

## Input Sanitization

Never trust user input before storing or displaying:

- **Trim whitespace** from all string inputs before storing
- **Validate length** to prevent database overflow
- **Escape special characters** before using in dynamic XPath or query strings
- **Validate file uploads** — check MIME type and size limits in Java actions

```
// Mendix expression: trim before set
trim($Input/Name)

// Check length
if length($Input/Description) > 500 then 
    'Description cannot exceed 500 characters.'
else
    ''
```

---

## Audit Trail Rules

- Every entity that holds business data must have:
  - `CreatedDate` (DateTime, auto-set)
  - `CreatedBy` (String or association to User)
  - `LastModifiedDate` (DateTime, auto-updated)
  - `LastModifiedBy` (String or association to User)
- Use a `SUB_Audit_SetTimestamps($Object)` sub-microflow to stamp these fields consistently

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Define access rules for every entity | Leave entities with default (open) access |
| Validate all inputs server-side | Rely only on client-side (page) validation |
| Apply XPath constraints for user-scoped data | Return all records and filter in-memory |
| Store credentials in encrypted constants | Hardcode API keys in microflows/JS actions |
| Log unauthorized access attempts | Let security violations go unlogged |
| Restrict microflow access per module role | Leave all microflows open to all roles |
| Use HTTPS for all external API calls | Allow HTTP connections |

---

## Example Usage

> **Scenario:** Securing the Leave Approval feature

```
Security Setup:
  Entity: LeaveRequest
    Employee role: Read own, Create, Update (Draft only)
    HR_Manager role: Read all, Update, Delete
    Admin role: Full access

  Page: LeaveRequest_Overview
    Allowed roles: Employee, HR_Manager, Admin

  ACT_LeaveRequest_Approve:
    Allowed roles: HR_Manager, Admin only

  Validation:
    VAL_LeaveRequest_Validate called before ACT_LeaveRequest_Submit
    Server-side check: Employee = CurrentUser before allowing submit
```
