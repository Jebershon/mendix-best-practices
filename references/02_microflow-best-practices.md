# Skill 02 — Microflow Best Practices

**Purpose:** Define standards for building reliable, maintainable, and performant microflows including error handling, logging, sub-flow design, and transaction control.

---

## When to Use Microflow vs Nanoflow

| Criteria | Microflow | Nanoflow |
|----------|-----------|----------|
| Needs database access (retrieve/commit/rollback) | ✅ | ❌ |
| Called from server-side or scheduled events | ✅ | ❌ |
| Needs Java or REST/SOAP calls | ✅ | ❌ |
| Pure UI interaction (show/hide, set attribute) | ❌ | ✅ |
| Must work offline | ❌ | ✅ |
| Performance-critical client-side logic | ❌ | ✅ |
| Needs transaction control | ✅ | ❌ |

---

## Structural Rules

1. **Every microflow must have a single, clear responsibility.**
2. **Limit microflow length** — if it exceeds ~15 activities, extract sub-microflows.
3. **All microflows that modify data must use error handling (TRY-CATCH pattern).**
4. **Never hardcode values** — use constants or parameters.
5. **Always return a result** from ACT_ flows (Boolean success/failure or object).
6. **Add a description** to every microflow explaining its purpose.

---

## TRY-CATCH Error Handling Pattern (Mandatory)

All microflows that commit, delete, call external services, or execute Java actions **must** follow this pattern:

```
[Start]
    │
    ▼
[Try Block Start — custom error handler]
    │
    ├── Retrieve / Validate / Business Logic
    │
    ├── Commit Object(s)
    │
    ├── SUB_Log_Info("ACT_Employee_Create", "Employee created: " + $Employee/Name)
    │
    └── [End: return True]
    
[Error Handler (Catch)]
    │
    ├── SUB_Log_Error("ACT_Employee_Create", $latestError/message)
    │
    ├── Rollback (if needed)
    │
    └── [End: return False or Show error message]
```

**Mendix error handler configuration:**
- Set "Custom without rollback" on the error handler activity
- Use `$latestError` object to retrieve message and stacktrace
- Always log `$latestError/message` and `$latestError/stackTrace`

---

## Sub-Microflow Design

Use sub-microflows (`SUB_`) to encapsulate reusable logic. Rules:

- A `SUB_` flow should accept input parameters and return a clear output
- Never call UI actions (show page, show message) from a `SUB_` flow
- Keep `SUB_` flows stateless where possible
- Name them with the domain and action: `SUB_Employee_FormatFullName`

**Good sub-microflow candidates:**
- Formatting logic (date formatting, name concatenation)
- Repeated validation checks
- Notification dispatch
- Audit trail recording

---

## Logging Strategy

Use a centralized `SUB_Log_Write` sub-microflow (see Skill 09) for all logging.

**Log inside microflows at these points:**

| Point | Level | Template |
|-------|-------|----------|
| Microflow start (for complex flows) | `INFO` | `"[ACT_X] Started. Input: " + $Param` |
| Successful commit | `INFO` | `"[ACT_X] Entity saved. ID: " + $Obj/ID` |
| Validation failure | `WARNING` | `"[VAL_X] Validation failed: " + reason` |
| External call | `INFO` | `"[INT_X] Calling endpoint: " + URL` |
| Catch block | `ERROR` | `"[ACT_X] Error: " + $latestError/message` |

---

## Transaction Handling

- **Commit once** at the end of a flow, not after every change
- Use **rollback** in the error handler when multiple objects are modified
- Never call `Commit` inside a loop — batch commits outside the loop
- For bulk operations, use a list + single commit at the end

---

## Performance Rules

- **Retrieve with XPath** — always constrain retrieves with XPath, never retrieve all
- **Limit list iteration** — avoid retrieving large lists into memory
- **Avoid nested loops** — refactor with association-based retrieves
- **Use Database retrieves** (not In Memory) unless object is already in context
- **Cache constants** — do not retrieve constant entities on every call

---

## Microflow Template (Standard Structure)

```
Microflow: ACT_{Entity}_{Action}
Parameters:
  - $InputObject : {Entity}
  
[1. Input Validation]
  → Call VAL_{Entity}_{Check}($InputObject)
  → If false: Show error / return false

[2. Business Logic]
  → Change object / create related objects
  → Apply calculations

[3. Error-Handled Commit]
  → Try:
      Commit $InputObject
      SUB_Log_Write(INFO, "ACT_X", "Saved: " + $InputObject/Name)
      Return true
  → Catch:
      SUB_Log_Write(ERROR, "ACT_X", $latestError/message)
      Rollback
      Return false

[4. Post-Action]
  → Trigger notification if needed (call SUB_Notification_Send)
  → Return result to caller
```

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Always wrap commits in error handlers | Commit without error handling |
| Use sub-microflows for repeated logic | Copy-paste the same logic block |
| Log at meaningful points | Log every single activity (noise) |
| Use parameters — avoid global variables | Rely on session variables for business logic |
| Keep flows under 15 activities | Build monolithic flows with 40+ activities |
| Add a microflow description | Leave description empty |

---

## Example Usage

> **Scenario:** Create a new Leave Request

```
ACT_LeaveRequest_Create(LeaveRequest $Request)
  │
  ├── VAL_LeaveRequest_CheckBalance($Request)  → if false, return error
  ├── Set $Request/Status = "Pending"
  ├── [Error Handler]
  │     Try: Commit $Request
  │          SUB_Log_Write(INFO, "ACT_LeaveRequest_Create", "Created: " + $Request/ID)
  │          Return true
  │     Catch: SUB_Log_Write(ERROR, ..., $latestError/message)
  │            Rollback
  │            Return false
  └── SUB_Notification_SendApproval($Request)
```
