# Skill 04 — Java Action Development Guidelines

**Purpose:** Define standards for developing Java Actions in Mendix including when to use them, input validation, exception handling, logging, and performance best practices.

---

## When to Use Java Actions

Use Java Actions (`JA_`) only when microflows and built-in Mendix functions **cannot** accomplish the task:

| Use Case | Java Action? |
|----------|-------------|
| Cryptography / hashing / encoding | ✅ Yes |
| Complex string manipulation not available in Mendix | ✅ Yes |
| Third-party Java library integration | ✅ Yes |
| File processing (PDF generation, ZIP, Excel) | ✅ Yes |
| System-level operations (environment vars, processes) | ✅ Yes |
| Simple REST call | ❌ Use Mendix REST Call activity |
| Database retrieve/commit | ❌ Use microflow |
| Basic string operations | ❌ Use Mendix expressions |
| Date formatting | ❌ Use Mendix formatDateTime() |

---

## Java Action Naming

Format: `JA_{Domain}_{Action}`

Examples:
- `JA_Crypto_HashSHA256`
- `JA_PDF_GenerateFromTemplate`
- `JA_File_CompressToZip`
- `JA_Util_ParseJSON`

---

## Input Validation

**Always validate inputs before processing.** Never trust that Mendix will send non-null/non-empty values.

```java
// Validate required string parameter
if (inputString == null || inputString.isEmpty()) {
    throw new CoreException("JA_Crypto_HashSHA256: inputString cannot be null or empty.");
}

// Validate required object parameter
if (inputObject == null) {
    throw new CoreException("JA_PDF_Generate: inputObject cannot be null.");
}

// Validate numeric range
if (pageSize <= 0 || pageSize > 1000) {
    throw new CoreException("JA_File_Process: pageSize must be between 1 and 1000.");
}
```

---

## Exception Handling

- **Never swallow exceptions silently.**
- Always wrap main logic in try-catch.
- Re-throw as `CoreException` so Mendix error handlers can catch them.
- Include context in the error message (action name + parameter details).

```java
@Override
public String executeAction() throws Exception {
    // --- Validation ---
    if (this.InputText == null || this.InputText.isEmpty()) {
        throw new CoreException("JA_Crypto_HashSHA256: InputText is required.");
    }

    // --- Main Logic ---
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(this.InputText.getBytes(StandardCharsets.UTF_8));
        return Base64.getEncoder().encodeToString(hash);

    } catch (NoSuchAlgorithmException e) {
        Core.getLogger(LOGNODE).error("JA_Crypto_HashSHA256: Algorithm not found. " + e.getMessage());
        throw new CoreException("JA_Crypto_HashSHA256 failed: " + e.getMessage(), e);
    } catch (Exception e) {
        Core.getLogger(LOGNODE).error("JA_Crypto_HashSHA256: Unexpected error. " + e.getMessage());
        throw new CoreException("JA_Crypto_HashSHA256 unexpected error: " + e.getMessage(), e);
    }
}
```

---

## Logging (Using Mendix Core Logger)

Define the log node as a constant at the top of the action class:

```java
private static final String LOGNODE = "YourModuleName";
```

Use structured log messages:

```java
// INFO: major milestones
Core.getLogger(LOGNODE).info("JA_PDF_Generate: Generating for EmployeeID=" + employeeId);

// DEBUG: verbose details (only in dev/staging)
Core.getLogger(LOGNODE).debug("JA_PDF_Generate: Template loaded: " + templateName);

// WARN: non-critical issues
Core.getLogger(LOGNODE).warn("JA_PDF_Generate: Template not found, using default.");

// ERROR: failures
Core.getLogger(LOGNODE).error("JA_PDF_Generate: Failed. Error: " + e.getMessage());
```

---

## Standard Java Action Template

```java
// BEGIN EXTRA CODE
private static final String LOGNODE = "HR_Module";
// END EXTRA CODE

@Override
public {ReturnType} executeAction() throws Exception {

    // --- BEGIN USER CODE ---

    // 1. Input Validation
    if (this.InputParam == null || this.InputParam.isEmpty()) {
        throw new CoreException("JA_{Domain}_{Action}: InputParam is required.");
    }

    // 2. Log Start
    Core.getLogger(LOGNODE).info("JA_{Domain}_{Action}: Started. Input=" + this.InputParam);

    // 3. Main Logic
    try {
        // Your logic here
        String result = processInput(this.InputParam);

        // 4. Log Success
        Core.getLogger(LOGNODE).info("JA_{Domain}_{Action}: Completed successfully.");

        return result;

    } catch (SpecificException e) {
        Core.getLogger(LOGNODE).error("JA_{Domain}_{Action}: SpecificException - " + e.getMessage());
        throw new CoreException("JA_{Domain}_{Action} failed: " + e.getMessage(), e);

    } catch (Exception e) {
        Core.getLogger(LOGNODE).error("JA_{Domain}_{Action}: Unexpected - " + e.getMessage());
        throw new CoreException("JA_{Domain}_{Action} unexpected error: " + e.getMessage(), e);
    }

    // --- END USER CODE ---
}
```

---

## Performance Optimization

- **Close all streams and resources** in a `finally` block or use try-with-resources
- **Avoid loading entire files into memory** — use streaming for large files
- **Reuse heavy objects** (e.g., MessageDigest, PDF templates) via static initialization if thread-safe
- **Do not use Thread.sleep()** — use Mendix scheduled events for delays
- **Limit use of reflection** — it is slow and brittle

```java
// ✅ Correct: Try-with-resources (auto-closes stream)
try (InputStream is = new FileInputStream(filePath)) {
    // process stream
}

// ❌ Wrong: Stream not closed on exception
InputStream is = new FileInputStream(filePath);
processStream(is);
is.close();
```

---

## Error-Safe Return Patterns

When a Java Action should not throw (returns Boolean for success/failure):

```java
@Override
public Boolean executeAction() throws Exception {
    try {
        // main logic
        return true;
    } catch (Exception e) {
        Core.getLogger(LOGNODE).error("JA_X_Y: " + e.getMessage());
        // Do NOT re-throw — return false so Mendix can handle gracefully
        return false;
    }
}
```

When a Java Action MUST signal failure to Mendix error handler:
```java
// Re-throw as CoreException — caught by Mendix TRY-CATCH in microflow
throw new CoreException("JA_X_Y failed: " + e.getMessage(), e);
```

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Validate all inputs before logic | Trust that inputs will never be null |
| Log at start, success, and error | Log inside every loop iteration |
| Use try-with-resources for I/O | Leave streams unclosed |
| Throw `CoreException` for Mendix to catch | Throw raw `RuntimeException` |
| Keep one Java Action per function | Build multi-purpose Java actions |
| Use `final` for constants | Use magic strings/numbers inline |

---

## Example Usage

> **Scenario:** Hash a password before storing it.

```java
// JA_Crypto_HashSHA256
// Input:  String plainText
// Output: String hashedValue

if (this.PlainText == null || this.PlainText.isEmpty()) {
    throw new CoreException("JA_Crypto_HashSHA256: PlainText cannot be empty.");
}
try {
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    byte[] hash = digest.digest(this.PlainText.getBytes(StandardCharsets.UTF_8));
    return Base64.getEncoder().encodeToString(hash);
} catch (Exception e) {
    throw new CoreException("JA_Crypto_HashSHA256 failed: " + e.getMessage(), e);
}
```

> **Called from microflow:**
```
ACT_Employee_SetPassword($Employee, $PlainPassword)
  → JA_Crypto_HashSHA256($PlainPassword) → $HashedPassword
  → Change $Employee/PasswordHash = $HashedPassword
  → Commit $Employee
```
