# Skill 11 — Performance Optimization Rules

**Purpose:** Define Mendix-specific performance patterns to minimize database load, reduce unnecessary retrievals, use batching, and ensure efficient queries.

---

## Core Performance Principles

1. **Fetch only what you need** — always constrain retrievals with XPath
2. **Minimize round-trips** — batch operations, avoid loops with inner retrievals
3. **Commit once** — accumulate changes, commit at the end
4. **Avoid memory retrieves for large datasets** — use database retrievals with limits
5. **Cache when safe** — avoid re-retrieving static or session-scoped data

---

## Retrieval Optimization

### Always Use XPath Constraints

```
// ❌ Wrong: Retrieve ALL employees, filter in memory
Retrieve all Employee objects
→ Loop → If $Employee/IsActive = true → process

// ✅ Right: Filter at database level
Retrieve Employee WHERE [IsActive = true and Department = $Dept]
```

### Limit Result Sets

```
// ❌ Wrong: Retrieve unlimited — risk of OOM on large datasets
Retrieve ALL LeaveRequests (no limit)

// ✅ Right: Always set a limit and use sort
Retrieve LeaveRequests
  XPath: [Status = 'Pending']
  Limit: 500
  Sort: SubmittedDate DESC
```

### Use Association Traversal Instead of Double Retrieves

```
// ❌ Wrong: Two separate database calls
$Employee = Retrieve Employee WHERE [ID = $EmployeeID]
$Department = Retrieve Department WHERE [ID = $Employee/DepartmentID]

// ✅ Right: Traverse association in one step
$Employee = Retrieve Employee WHERE [ID = $EmployeeID]
$DeptName = $Employee/Employee_Department/Department/Name
```

---

## Loop Optimization

### Never Retrieve Inside a Loop

```
// ❌ Wrong: N+1 query problem — one DB call per employee
Loop over $EmployeeList:
  → Retrieve $Requests WHERE [Employee = $CurrentEmployee]  ← DB call per iteration
  → Process $Requests

// ✅ Right: Retrieve all at once, associate in memory
$AllRequests = Retrieve LeaveRequest WHERE [Status = 'Pending']
Loop over $EmployeeList:
  → Filter $AllRequests WHERE Employee = $CurrentEmployee  (in memory)
  → Process filtered list
```

### Batch Commits — Never Commit Inside a Loop

```
// ❌ Wrong: Commit on every iteration
Loop over $Employees:
  → Change $Employee/Status = 'Active'
  → Commit $Employee   ← one DB write per employee

// ✅ Right: Commit all at end
Loop over $Employees:
  → Change $Employee/Status = 'Active'   (in memory only)
End loop
→ Commit $EmployeeList  (single batch commit)
```

---

## Database vs In-Memory Retrieval

| Scenario | Use |
|----------|-----|
| First retrieval of an object in a flow | **Database** |
| Object already retrieved earlier in same flow | **In Memory / Association** |
| Filtering a large dataset | **Database with XPath** |
| Filtering a small already-retrieved list | **In Memory** |
| Checking existence of one record | **Database with Count or First** |

```
// Check existence efficiently — retrieve one, not all
$Exists = Retrieve first LeaveRequest WHERE [Employee = $Emp and Status = 'Pending']
→ If $Exists != empty → "Already has pending request"
```

---

## Avoiding Unnecessary Object Creation

- **Do not create objects speculatively** — only create when you are sure you'll commit
- **Use Change Object instead of New + Delete** when updating existing data
- **Rollback uncommitted objects** in error handlers to prevent memory leaks

---

## Aggregation Optimization

```
// ❌ Wrong: Retrieve all records and count in a loop
$All = Retrieve ALL LeaveRequest
$Count = 0
Loop: $Count = $Count + 1

// ✅ Right: Use aggregate function
$Count = Count(LeaveRequest WHERE [Status = 'Pending' and Employee = $Emp])
$TotalDays = Sum(LeaveRequest/Days WHERE [Year = 2024 and Employee = $Emp])
```

---

## Scheduled Event & Bulk Processing

For nightly jobs or bulk operations processing thousands of records:

1. **Use pagination** — process in chunks of 500–1000
2. **Log progress** — `SUB_Log_Write(INFO, ..., "Processed batch " + $BatchNum)`
3. **Commit per batch** — not per record, not all at once
4. **Track failures independently** — don't let one bad record abort the entire batch

```
ACT_Scheduled_SyncEmployees
  │
  ├── $Offset = 0, $BatchSize = 500, $TotalProcessed = 0
  │
  ├── [Loop]:
  │     $Batch = Retrieve Employees LIMIT $BatchSize OFFSET $Offset
  │     If $Batch is empty → exit loop
  │     
  │     [Process each in $Batch]
  │     Commit $Batch
  │     $TotalProcessed = $TotalProcessed + length($Batch)
  │     $Offset = $Offset + $BatchSize
  │
  └── SUB_Log_Write(INFO, 'ACT_Scheduled_SyncEmployees', 
                    'Sync complete. Total: ' + $TotalProcessed)
```

---

## Page & UI Performance

- **Limit data grids to 20–50 rows** — use pagination, never load all
- **Use lazy loading** for associations on detail pages — only fetch what's visible
- **Avoid loading related lists on page open** — use on-demand buttons instead
- **Minimize microflow calls from pages** — consolidate into one data-fetching microflow

---

## XPath Query Optimization Tips

```
// ✅ Efficient: Index-friendly equality check
[Status = 'Pending']

// ✅ Efficient: Association filter
[Employee_LeaveRequest/HR.Employee/IsActive = true]

// ⚠️ Use carefully: Functions on attributes prevent index use
[substring(Name, 1, 3) = 'Ali']   ← not index-friendly

// ✅ Better: Use contains if full-text search needed
[contains(Name, 'Ali')]            ← still do a DB contains, but clearer intent
```

---

## Caching Patterns

- **System constants** — retrieved once at startup, no runtime retrieval needed
- **Enumeration lists** — never retrieved from DB; use enum values directly in XPath
- **Session-scoped lookups** — retrieve once at login, store in session variable
- **Reference data (departments, roles)** — retrieve once at page load, pass as parameter to sub-flows

```
// ❌ Wrong: Retrieve department name in every sub-flow
SUB_Format_EmployeeInfo($Employee):
  $Dept = Retrieve Department WHERE [ID = $Employee/DeptID]  ← DB hit every time
  return $Employee/Name + ' (' + $Dept/Name + ')'

// ✅ Right: Pass already-retrieved data as parameter
SUB_Format_EmployeeInfo($Employee, $DeptName):
  return $Employee/Name + ' (' + $DeptName + ')'
```

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Always constrain XPath with meaningful filters | Retrieve all records and filter in memory |
| Commit at the end of a batch, not inside loops | Commit inside every loop iteration |
| Use Count/Sum aggregates instead of loops | Loop to count or sum large datasets |
| Paginate bulk operations in chunks of ≤1000 | Process millions of records in one pass |
| Use association traversal instead of re-retrieving | Retrieve the same object multiple times |
| Limit data grid rows to 20–50 with pagination | Load all records in one grid |
| Pass already-retrieved objects as parameters | Re-retrieve objects that are already in scope |

---

## Example Usage

> **Scenario:** Year-end leave balance reset for all active employees

```
ACT_LeaveBalance_YearEndReset
  │
  ├── $TotalEmployees = Count(Employee WHERE [IsActive = true])
  ├── SUB_Log_Write(INFO, ..., 'Year-end reset starting. Target: ' + $TotalEmployees)
  │
  ├── $Offset = 0, $BatchSize = 200
  │
  ├── [Loop while batch not empty]:
  │     $Batch = Retrieve Employee WHERE [IsActive = true] LIMIT 200 OFFSET $Offset
  │     If empty → break
  │     
  │     Loop $Batch:
  │       Change $Emp/AnnualLeaveBalance = CONST_HR_DefaultLeaveBalance
  │     
  │     Commit $Batch
  │     SUB_Log_Write(INFO, ..., 'Batch committed. Offset: ' + $Offset)
  │     $Offset = $Offset + 200
  │
  └── SUB_Log_Write(INFO, ..., 'Year-end reset complete.')
```
