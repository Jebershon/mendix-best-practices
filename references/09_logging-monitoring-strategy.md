# Skill 09 — Logging & Monitoring Strategy

**Purpose:** Define a consistent, traceable logging strategy across all microflows, Java actions, and integrations. Includes a dedicated log entity, log viewer page, and correlation ID tracking.

---

## Logging Philosophy

- **Log what matters** — start events, key decisions, errors, integration calls
- **Never log PII** (names, IDs, passwords, tokens in plain text)
- **Use structured messages** — always include module + flow name + context
- **Make logs actionable** — an ERROR log must include enough info to diagnose the issue without source access

---

## Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| `INFO` | Normal flow milestones, successful operations | "Employee record created: EMP-2045" |
| `DEBUG` | Detailed tracing during dev/test (disable in prod) | "XPath filter applied: Status=Pending" |
| `WARNING` | Non-fatal issues that need attention | "Fallback value used for missing config" |
| `ERROR` | Failures that need immediate investigation | "Oracle API call failed: HTTP 500" |
| `CRITICAL` | System-breaking issues | "Database connection lost" |

**Production rule:** Set log level to `INFO` in production. Use `DEBUG` only in development/staging.

---

## Log Message Format

```
[{LogNode}] [{FlowName}] [{CorrelationID}] {Message}
```

Examples:
```
[HR_Module] [ACT_LeaveRequest_Submit] [CID-20240501-001] Started. EmployeeID=EMP-2045
[HR_Module] [INT_Oracle_SyncEmployee] [CID-20240501-001] HTTP 200. Records synced: 12
[HR_Module] [ACT_LeaveRequest_Submit] [CID-20240501-001] ERROR: Validation failed. Missing EndDate.
```

---

## Persistent Log Entity

Create a `MicroflowLog` entity in the `Logging` domain model to persist all important execution records.

### Entity: `MicroflowLog`

| Attribute | Type | Description |
|-----------|------|-------------|
| `LogID` | AutoNumber | Unique record ID |
| `CorrelationID` | String (100) | Unique trace ID for the full request |
| `FlowName` | String (200) | Name of the microflow/Java action |
| `LogLevel` | Enumeration | INFO / DEBUG / WARNING / ERROR / CRITICAL |
| `Message` | String (2000) | Human-readable log message |
| `ErrorDetails` | String (unlimited) | Full error message + stack trace (on error) |
| `InputData` | String (2000) | Sanitized input summary (no PII) |
| `ExecutedBy` | String (200) | Username or system role |
| `ExecutedAt` | DateTime | Timestamp (auto-set) |
| `ModuleName` | String (100) | Mendix module name |
| `DurationMs` | Integer | Execution duration in milliseconds (optional) |
| `IsError` | Boolean | Quick filter flag for errors |

### Enumeration: `ELogLevel`

```
ELogLevel:
  - DEBUG
  - INFO
  - WARNING
  - ERROR
  - CRITICAL
```

---

## Centralized Logging Sub-Microflow

Create `SUB_Log_Write` as the **single entry point for all logging** in the application.

### SUB_Log_Write (Parameters)

| Parameter | Type | Required |
|-----------|------|---------|
| `LogLevel` | ELogLevel | ✅ |
| `FlowName` | String | ✅ |
| `Message` | String | ✅ |
| `CorrelationID` | String | Optional |
| `ErrorDetails` | String | Optional |
| `InputData` | String | Optional |

### SUB_Log_Write (Logic)

```
SUB_Log_Write($LogLevel, $FlowName, $Message, $CorrelationID, $ErrorDetails, $InputData)
  │
  ├── Create new MicroflowLog object
  ├── Set: LogLevel = $LogLevel
  ├── Set: FlowName = $FlowName
  ├── Set: Message = $Message
  ├── Set: CorrelationID = if $CorrelationID != '' then $CorrelationID else 'NO-CID'
  ├── Set: ErrorDetails = $ErrorDetails
  ├── Set: InputData = $InputData
  ├── Set: ExecutedBy = $currentUser/Name (or 'System' if no session)
  ├── Set: ExecutedAt = [%CurrentDateTime%]
  ├── Set: ModuleName = 'HR_Module'   ← constant per module
  ├── Set: IsError = ($LogLevel = ELogLevel.ERROR or $LogLevel = ELogLevel.CRITICAL)
  ├── Commit MicroflowLog (WITHOUT EVENTS)
  │
  └── Also write to Mendix application log:
        Core.getLogger('HR_Module').{level}('[{FlowName}] [{CorrelationID}] {Message}')
```

**Important:** Commit MicroflowLog WITHOUT rollback — even if the parent transaction fails, the log must persist.

---

## Correlation ID Strategy

A **Correlation ID** (CID) ties together all log entries for a single user request or business transaction.

### Generating a Correlation ID

```
SUB_Log_GenerateCID()
  → $CID = 'CID-' + formatDateTime([%CurrentDateTime%], 'yyyyMMdd-HHmmss') + '-' + $RandomSuffix
  → Return $CID
```

Or use a UUID Java action: `JA_Util_GenerateUUID()` → returns a UUID string.

### Usage Pattern

```
ACT_LeaveRequest_Submit($Request)
  │
  ├── $CID = SUB_Log_GenerateCID()
  │
  ├── SUB_Log_Write(INFO, 'ACT_LeaveRequest_Submit', 'Started', $CID, '', 'RequestID=' + $Request/ID)
  │
  ├── [Business Logic]
  │
  ├── SUB_Log_Write(INFO, 'ACT_LeaveRequest_Submit', 'Submitted successfully', $CID)
  │
  └── [Error Handler]
        SUB_Log_Write(ERROR, 'ACT_LeaveRequest_Submit', 'Failed: ' + $latestError/message, $CID, $latestError/stackTrace)
```

---

## Where to Log

| Location | Log Points |
|----------|-----------|
| ACT_ microflows | Start (INFO), success (INFO), error (ERROR) |
| INT_ microflows | Before call (INFO), after call (INFO), error (ERROR) |
| VAL_ microflows | Validation failure (WARNING) |
| Java Actions | Key operations (INFO), errors (ERROR) |
| Scheduled events | Start (INFO), completion with counts (INFO), errors (ERROR) |
| Login/security events | Failed auth (WARNING), unauthorized access (WARNING/ERROR) |

---

## Log Viewer Page (Admin)

Create a page `Admin_MicroflowLog_Overview` accessible only to the `Admin` role.

### Page Configuration

- **Title:** Microflow Execution Logs
- **Access:** Admin role only
- **Layout:** Back-office layout with sidebar

### Data Grid Columns

| Column | Width | Sortable | Filterable |
|--------|-------|----------|-----------|
| ExecutedAt | 180 | ✅ (default DESC) | ✅ (date range) |
| LogLevel | 100 | ✅ | ✅ (dropdown) |
| FlowName | 250 | ✅ | ✅ (contains) |
| Message | 400 | ❌ | ✅ (contains) |
| CorrelationID | 200 | ✅ | ✅ (equals) |
| ExecutedBy | 150 | ✅ | ✅ (contains) |
| IsError | 80 | ✅ | ✅ (boolean) |

### Filters

- Log Level (multi-select): DEBUG / INFO / WARNING / ERROR / CRITICAL
- Date range: From — To
- Is Error: Yes / No / All
- Flow Name: Contains text search
- Correlation ID: Exact match

### Detail Popup

On row click, show a popup with full details:
- All attributes displayed
- `ErrorDetails` in a read-only text area (full stack trace)
- `InputData` in a read-only text area

---

## Log Retention Policy

- **DEBUG logs:** Purge after 7 days
- **INFO logs:** Purge after 30 days
- **WARNING logs:** Purge after 90 days
- **ERROR/CRITICAL logs:** Keep for 1 year minimum

Create a **scheduled microflow** `ACT_Log_Purge` to run nightly and delete old records.

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Use `SUB_Log_Write` for all logging | Write log logic inline in every microflow |
| Include Correlation ID on all flows | Log without any trace context |
| Commit log records without events/rollback | Let logs get rolled back with parent transactions |
| Log errors with full stack trace | Log only "An error occurred" |
| Restrict log viewer page to Admin | Allow all users to see logs |
| Purge old logs on a schedule | Let logs grow unbounded in the database |

---

## Example Usage

> **Scenario:** Integration call with full trace

```
INT_OracleFusion_GetEmployee($EmployeeID, $CID)
  │
  ├── SUB_Log_Write(INFO, 'INT_OracleFusion_GetEmployee', 
  │                 'Calling Oracle. EmployeeID=' + $EmployeeID, $CID)
  │
  ├── [REST Call to Oracle Fusion API]
  │
  ├── If HTTP 200:
  │     SUB_Log_Write(INFO, 'INT_OracleFusion_GetEmployee', 
  │                   'Success. Records returned.', $CID)
  │     → Parse and return object
  │
  └── If Error:
        SUB_Log_Write(ERROR, 'INT_OracleFusion_GetEmployee',
                      'Oracle call failed: ' + $HTTPStatus, $CID,
                      $ResponseBody)
        → Return empty
```
