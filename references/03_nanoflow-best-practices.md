# Skill 03 — Nanoflow Best Practices

**Purpose:** Define standards for nanoflow development with a focus on UI logic, offline capability, safe microflow invocation, and performance.

---

## What Nanoflows Are For

Nanoflows run entirely on the client (browser or device). They are ideal for:

- **UI interactions** — show/hide widgets, toggle states, set attributes in memory
- **Client-side validations** — before committing or calling server logic
- **Offline-first logic** — all operations in offline-capable apps
- **Lightweight navigation** — opening/closing pages, popups
- **Calling microflows** when server interaction is needed from the UI

---

## When to Use Nanoflow vs Microflow

| Use Case | Nanoflow | Microflow |
|----------|----------|-----------|
| Show a confirmation popup | ✅ | ❌ |
| Change an attribute without saving | ✅ | ❌ |
| Validate form before submission | ✅ | ❌ |
| Works in offline PWA / native app | ✅ | ❌ |
| Retrieve from database | ❌ | ✅ |
| Call REST/SOAP API | ❌ | ✅ |
| Execute complex business rules | ❌ | ✅ |
| Send email / trigger workflow | ❌ | ✅ |

---

## UI Logic Separation Rule

> **Never embed business logic inside a nanoflow.**

Nanoflows must only handle:
1. UI state changes
2. User input collection
3. Validation messages
4. Delegation to microflows for server-side work

**Wrong pattern:**
```
NF_Employee_Save
  → Change Attribute: Employee/Salary = calculation...
  → Complex XPath filtering
  → Commit Employee
```

**Right pattern:**
```
NF_Employee_Save
  → Show loading spinner
  → Call ACT_Employee_Save($Employee)   ← microflow handles logic
  → Hide loading spinner
  → Show success message
```

---

## Calling Microflows Safely from Nanoflows

When invoking a microflow from a nanoflow:

1. **Always capture the return value** and check for success/failure
2. **Show user feedback** before and after microflow execution (loading indicators)
3. **Do not assume success** — handle false/null return values explicitly
4. **Avoid chaining multiple microflow calls** in a single nanoflow — keep it to one server round-trip

```
NF_LeaveRequest_Submit($Request)
  │
  ├── [Show loading indicator: true]
  ├── [Call ACT_LeaveRequest_Create($Request) → $Success]
  ├── [Show loading indicator: false]
  ├── If $Success = true  → Show "Request submitted" / Close page
  └── If $Success = false → Show "Submission failed. Please try again."
```

---

## Offline Capability Rules

When building nanoflows for offline / native apps:

- Only use **Nano-compatible** activities (no database retrieve, no REST calls)
- Use **synchronize** actions carefully — they trigger a full sync
- Create/change objects will be stored locally until sync
- **Never call microflows** in an offline nanoflow (server unreachable)
- Use **JSA_ JavaScript actions** for device-level operations (GPS, camera)

**Offline-safe activities in nanoflows:**
- Create object (stored locally)
- Change object
- Show page / close page
- Show message
- JavaScript action calls

---

## Performance Considerations

- Keep nanoflows **short and focused** — single responsibility
- Avoid complex conditional branching — move logic to microflows
- Minimize attribute changes per nanoflow execution
- Avoid deep loops — they block the UI thread
- Use `NF_` prefix even for sub-nanoflows: `NF_SUB_Format_Date`

---

## Naming Standard

| Pattern | Example |
|---------|---------|
| `NF_{Entity}_{Action}` | `NF_Employee_ToggleEdit` |
| `NF_{Context}_{UIAction}` | `NF_Page_ShowConfirmPopup` |
| `NF_SUB_{Logic}` | `NF_SUB_ValidateFormFields` |

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Use nanoflows for UI state management | Use nanoflows for complex business logic |
| Delegate to microflows for server calls | Call multiple microflows sequentially |
| Check microflow return values | Assume every microflow call succeeds |
| Keep nanoflows under 10 activities | Build long multi-purpose nanoflows |
| Use for offline apps exclusively | Mix offline + online logic in one nanoflow |
| Show loading/feedback to users | Call server logic silently without feedback |

---

## Example Usage

> **Scenario:** Employee profile edit toggle with save

```
NF_Employee_EditAndSave($Employee)
  │
  ├── If $Employee/IsEditing = false:
  │     → Change: $Employee/IsEditing = true
  │     → Show "Edit mode enabled"
  │
  └── If $Employee/IsEditing = true:
        → [Show spinner]
        → Call ACT_Employee_Update($Employee) → $Result
        → [Hide spinner]
        → If $Result = true:
              Change: $Employee/IsEditing = false
              Show "Saved successfully"
          If $Result = false:
              Show "Save failed. Check your input."
```
