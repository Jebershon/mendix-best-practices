# Skill 12 — AI Code Generation Rules

**Purpose:** Define mandatory rules that AI (Claude or any assistant) must follow when generating Mendix code, logic, widget XML, Java actions, and JavaScript actions. These rules ensure all generated output is production-ready, clean, and maintainable.

---

## Non-Negotiable Generation Rules

The following rules apply to **every piece of code generated**, without exception. There are no overrides.

---

### Rule 1: Always Apply Naming Conventions

Every generated component must follow the naming convention from Skill 01:

| Type | Required Prefix | Example |
|------|----------------|---------|
| Action microflow | `ACT_` | `ACT_Employee_Create` |
| Sub-microflow | `SUB_` | `SUB_Email_FormatBody` |
| Validation microflow | `VAL_` | `VAL_LeaveRequest_Validate` |
| Integration microflow | `INT_` | `INT_OracleFusion_GetEmployee` |
| Nanoflow | `NF_` | `NF_Form_ToggleEditMode` |
| Java action | `JA_` | `JA_Crypto_HashSHA256` |
| JavaScript action | `JSA_` | `JSA_Browser_CopyToClipboard` |
| Widget ID | `com.{company}.{module}.{Name}` | `com.bahri.hr.EmployeeBadge` |

❌ AI must **never** generate flows named `Microflow1`, `NewJavaAction`, `Untitled`, or similar placeholders.

---

### Rule 2: Always Include Error Handling

Every generated microflow that performs any of the following **must** include a TRY-CATCH pattern:

- Commits an object
- Deletes an object
- Calls a Java action
- Calls a REST/SOAP service
- Calls an INT_ microflow

**Mandatory error handler structure:**
```
[Try]
  → Business logic
  → Commit
  → Log success (INFO)
  → Return true / result object

[Catch — Custom Error Handler]
  → Log error: SUB_Log_Write(ERROR, '{FlowName}', $latestError/message, $CID, $latestError/stackTrace)
  → Rollback
  → Return false / empty
```

❌ AI must **never** generate a commit without an error handler.

---

### Rule 3: Always Include Logging

Every generated microflow must include at minimum:

- **On error (ERROR level):** Always — error message + stack trace
- **On success for ACT_ flows (INFO level):** Summary of what was done
- **For INT_ flows (INFO level):** Before and after the external call

Use the centralized logger: `SUB_Log_Write(Level, FlowName, Message, CorrelationID, ErrorDetails)`

❌ AI must **never** generate a flow with no logging whatsoever.

---

### Rule 4: Always Validate Inputs

Every generated flow that accepts parameters must validate them before use:

**Microflow validation:**
```
If $InputObject = empty → Show error / Return false
If $InputString = ''    → Show error / Return false
If $InputDate = empty   → Show error / Return false
```

**Java action validation:**
```java
if (this.InputParam == null || this.InputParam.isEmpty()) {
    throw new CoreException("JA_X_Y: InputParam is required.");
}
```

**JavaScript action validation:**
```javascript
if (!inputParam || inputParam.trim() === '') {
    throw new Error('JSA_X_Y: inputParam is required.');
}
```

❌ AI must **never** generate code that accesses parameters without null/empty checks.

---

### Rule 5: Always Place Components in the Correct Folder

When generating Mendix logic, always specify the correct folder placement:

| Component | Target Folder |
|-----------|--------------|
| `ACT_` microflow | `microflows/actions/` |
| `SUB_` microflow | `microflows/sub/` |
| `VAL_` microflow | `microflows/validations/` |
| `INT_` microflow | `microflows/integrations/` |
| Nanoflow | `nanoflows/` |
| Java action | `java_actions/` |
| JavaScript action | `javascript_actions/` |
| Widget | `widgets/{WidgetName}/` |

❌ AI must **never** generate components without specifying their correct module folder.

---

### Rule 6: Never Generate Duplicate Logic

Before generating a new sub-microflow or utility, AI must check if equivalent functionality already exists:

- Check if a `SUB_Validate_{Field}` already exists before creating another
- Check if a notification pattern already exists in `Shared_Module`
- Check if an audit stamp sub-flow already exists

If duplication is suspected, **ask the user** rather than generating a new copy.

❌ AI must **never** generate copy-paste logic when a SUB_ pattern already exists.

---

### Rule 7: Widget XML Must Be Zero-Error

When generating widget XML:

- [ ] `id` attribute must match `package.json` `name` field exactly
- [ ] `pluginWidget="true"` must always be present
- [ ] `needsEntityContext` must be explicitly set (`"true"` or `"false"`)
- [ ] Every `attribute` property must have at least one `<attributeType>` child
- [ ] Every `enumeration` property must have a `defaultValue`
- [ ] Every `boolean` property must have a `defaultValue`
- [ ] Action properties must be `required="false"`
- [ ] Schema URL must be: `https://apidocs.mendix.com/pluggable-widgets/1.0/widget.xsd`

❌ AI must **never** generate widget XML without validating against this checklist.

---

### Rule 8: Never Hardcode Sensitive Values

Generated code must **never** contain:

- API keys or tokens inline
- Passwords or credentials
- Hardcoded environment URLs (use constants)
- Hardcoded role names as strings (use `[%UserRole_RoleName%]`)

❌ Anti-pattern:
```java
String apiKey = "sk-abc123xyz";  // NEVER
String url = "https://api.prod.oracle.com";  // NEVER
```

✅ Correct:
```
Use Mendix constant: CONST_Oracle_APIKey
Use Mendix constant: CONST_Oracle_BaseURL
```

---

### Rule 9: Always Use Async/Await in JavaScript Actions

Generated JavaScript actions must always use `async/await`:

❌ Never generate:
```javascript
fetch(url).then(r => r.json()).then(data => { ... }).catch(e => { ... });
```

✅ Always generate:
```javascript
try {
    const response = await fetch(url);
    const data = await response.json();
    return data;
} catch (error) {
    throw new Error(`JSA_X_Y failed: ${error.message}`);
}
```

---

### Rule 10: Add Descriptions to All Generated Components

Every generated microflow, Java action, nanoflow, and widget must include:

- A **description field** explaining its purpose
- **Parameter descriptions** for each input/output
- **Inline comments** for complex logic sections

---

### Rule 11: Never Over-Engineer

Generated code must be:

- **As simple as possible** for the stated requirement
- **No speculative features** — only what was asked for
- **No premature abstraction** — don't create interfaces for single-use logic
- **No unused parameters** or return values

---

### Rule 12: Security Rules Are Always Enforced

Generated code must:

- Always check user role/ownership before modifying data
- Always use XPath constraints for user-scoped data retrieves
- Never expose internal IDs in page URLs without server-side validation
- Always validate file uploads (type + size) in Java actions
- Always use HTTPS for external API calls

---

## Generation Quality Checklist

Before outputting any generated Mendix component, AI must verify:

```
Naming:
  [ ] Correct prefix applied
  [ ] Entity/domain context included in name
  [ ] No placeholder names (Microflow1, NewAction, etc.)

Error Handling:
  [ ] TRY-CATCH present for any commit/delete/integration
  [ ] Error log call in catch block
  [ ] Rollback in catch block

Logging:
  [ ] At least one SUB_Log_Write call present
  [ ] Error logging included in error handler

Validation:
  [ ] All parameters checked for null/empty
  [ ] Validation failure returns meaningful message

Security:
  [ ] No hardcoded credentials
  [ ] Access check if operating on user-scoped data

Placement:
  [ ] Correct folder specified for the component type

Documentation:
  [ ] Description field populated
  [ ] Parameters described
```

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Apply all 12 rules to every generated output | Skip rules "just this once" for speed |
| Name every component properly on first generation | Use placeholder names that need renaming |
| Ask the user if a SUB_ flow might already exist | Generate duplicate logic silently |
| Include error handling even for "simple" flows | Assume simple flows can't fail |
| Validate all parameters | Trust that inputs are always valid |
| Specify folder placement explicitly | Leave component placement unspecified |
| Include logging that aids debugging | Generate flows with zero logging |

---

## Example: Complete Generation Output

> **User Request:** "Generate a microflow to approve a leave request."

**AI must generate:**

```
Microflow: ACT_LeaveRequest_Approve
Folder: HR_Module/microflows/actions/
Allowed Roles: HR_Manager, Admin
Description: Approves a pending leave request, notifies the employee, and syncs to Oracle Fusion.

Parameters:
  - $Request (LeaveRequest): The request to approve
  - $ApproverNote (String): Optional comment from the approver

Logic:
  1. VAL Check:
     If $Request = empty → Log WARNING + Return false
     If $Request/Status != 'Pending' → Log WARNING "Already processed" + Return false

  2. [Error Handler - Custom]
     Try:
       $CID = SUB_Log_GenerateCID()
       SUB_Log_Write(INFO, 'ACT_LeaveRequest_Approve', 'Approving. ID=' + $Request/ID, $CID)
       
       Change $Request/Status = 'Approved'
       Change $Request/ApprovedDate = [%CurrentDateTime%]
       Change $Request/ApprovedBy = $currentUser/Name
       Change $Request/ApproverNote = $ApproverNote
       
       Commit $Request
       
       SUB_Log_Write(INFO, 'ACT_LeaveRequest_Approve', 'Approved successfully', $CID)
       SUB_Notification_Send('LeaveApproved', $Request/Employee/Email, 'Leave Approved', ...)
       INT_OracleFusion_PostLeave($Request, $CID)
       
       Return true
       
     Catch:
       SUB_Log_Write(ERROR, 'ACT_LeaveRequest_Approve',
                     $latestError/message, $CID, $latestError/stackTrace)
       Rollback
       Return false
```
