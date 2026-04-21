# Skill 05 — JavaScript Action Guidelines

**Purpose:** Define standards for building JavaScript Actions in Mendix including async handling, error management, API calls, and UI responsiveness.

---

## When to Use JavaScript Actions

Use JavaScript Actions (`JSA_`) when you need browser/device-level capabilities not available in microflows or nanoflows:

| Use Case | JS Action? |
|----------|-----------|
| Access browser APIs (clipboard, geolocation, localStorage) | ✅ Yes |
| DOM manipulation or custom events | ✅ Yes |
| Third-party JavaScript SDK integration | ✅ Yes |
| Client-side file reading/manipulation | ✅ Yes |
| Calling external APIs directly from client | ✅ Yes (with caution) |
| Mendix database operations | ❌ Use microflow |
| Complex business rules | ❌ Use microflow |
| UI state toggling | ❌ Use nanoflow |
| Simple string formatting | ❌ Use Mendix expression |

---

## Naming Convention

Format: `JSA_{Context}_{Action}`

Examples:
- `JSA_Browser_CopyToClipboard`
- `JSA_Device_GetCurrentLocation`
- `JSA_Storage_ReadLocalValue`
- `JSA_API_FetchExchangeRate`

---

## Standard Template

Every JavaScript Action must follow this structure:

```javascript
// JSA_{Context}_{Action}
// Description: What this action does
// Parameters:
//   - inputParam (String): Description
// Returns: String | Boolean | Object

/**
 * @param {string} inputParam
 * @returns {Promise<string>}
 */
export async function JSA_Context_Action(inputParam) {
    // 1. Input Validation
    if (!inputParam || inputParam.trim() === "") {
        throw new Error("JSA_Context_Action: inputParam is required.");
    }

    try {
        // 2. Main Logic
        const result = await performOperation(inputParam);

        // 3. Return result
        return result;

    } catch (error) {
        // 4. Error Handling — never swallow
        console.error(`JSA_Context_Action failed: ${error.message}`, error);
        throw new Error(`JSA_Context_Action failed: ${error.message}`);
    }
}
```

---

## Async / Await Best Practices

- **Always use `async/await`** — never use `.then().catch()` chains (harder to read/debug)
- **Always `await` Promises** — never leave floating promises
- **Never use `setTimeout` for logic flow** — use proper async patterns

```javascript
// ✅ Correct: async/await with error handling
export async function JSA_API_FetchData(url) {
    if (!url) throw new Error("JSA_API_FetchData: URL is required.");
    try {
        const response = await fetch(url, {
            method: "GET",
            headers: { "Content-Type": "application/json" }
        });
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        return await response.json();
    } catch (error) {
        console.error("JSA_API_FetchData error:", error);
        throw new Error(`JSA_API_FetchData failed: ${error.message}`);
    }
}

// ❌ Wrong: .then chains — harder to trace errors
fetch(url).then(res => res.json()).then(data => data).catch(e => console.log(e));
```

---

## API Handling

When calling external APIs from a JavaScript Action:

1. **Always validate the URL parameter** before making the call
2. **Check HTTP response status** — do not assume `200 OK`
3. **Set a timeout** to avoid hanging the UI
4. **Never expose API keys** in JS actions — pass them as parameters from constants
5. **Parse response safely** — use try-catch around `JSON.parse()`

```javascript
export async function JSA_API_Post(endpoint, payload, apiKey) {
    if (!endpoint) throw new Error("JSA_API_Post: endpoint is required.");
    if (!payload)  throw new Error("JSA_API_Post: payload is required.");

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000); // 10s timeout

    try {
        const response = await fetch(endpoint, {
            method: "POST",
            headers: {
                "Content-Type": "application/json",
                "Authorization": `Bearer ${apiKey}`
            },
            body: JSON.stringify(payload),
            signal: controller.signal
        });

        clearTimeout(timeout);

        if (!response.ok) {
            const errBody = await response.text();
            throw new Error(`HTTP ${response.status}: ${errBody}`);
        }

        return await response.json();

    } catch (error) {
        clearTimeout(timeout);
        if (error.name === "AbortError") {
            throw new Error("JSA_API_Post: Request timed out.");
        }
        throw new Error(`JSA_API_Post failed: ${error.message}`);
    }
}
```

---

## Avoiding UI Blocking

JavaScript runs on the main browser thread. Long-running synchronous code freezes the UI.

- **Always use `async/await`** for I/O (fetch, file read, etc.)
- **Avoid large synchronous loops** — break into smaller chunks or use `setTimeout` with 0ms for yielding
- **Do not block for more than ~16ms** on the main thread
- For heavy computation, consider Web Workers (advanced)

```javascript
// ✅ Non-blocking: async fetch
const data = await fetch(url).then(r => r.json());

// ❌ Blocking: synchronous XHR — never use
const xhr = new XMLHttpRequest();
xhr.open("GET", url, false); // false = synchronous
xhr.send();
```

---

## Browser API Patterns

### Clipboard

```javascript
export async function JSA_Browser_CopyToClipboard(text) {
    if (!text) throw new Error("JSA_Browser_CopyToClipboard: text is required.");
    try {
        await navigator.clipboard.writeText(text);
        return true;
    } catch (error) {
        console.error("Clipboard write failed:", error);
        return false;
    }
}
```

### Geolocation

```javascript
export async function JSA_Device_GetLocation() {
    return new Promise((resolve, reject) => {
        if (!navigator.geolocation) {
            reject(new Error("Geolocation not supported by this browser."));
            return;
        }
        navigator.geolocation.getCurrentPosition(
            (pos) => resolve(`${pos.coords.latitude},${pos.coords.longitude}`),
            (err) => reject(new Error(`Geolocation error: ${err.message}`))
        );
    });
}
```

---

## Error Handling Rules

- Always re-throw errors so Mendix nanoflow/microflow error handlers can catch them
- Use `console.error()` for client-side debug output
- Include the action name in every error message for traceability
- Never return `null` silently on failure — throw a meaningful error

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Use `async/await` for all async operations | Use `.then().catch()` chains |
| Validate inputs at the top of every action | Trust inputs are always valid |
| Set timeouts on network requests | Let requests hang indefinitely |
| Re-throw errors for Mendix to catch | Swallow errors with empty catch |
| Keep each JS action single-purpose | Build multi-purpose JS actions |
| Use parameters for API keys | Hardcode secrets in JS code |

---

## Example Usage

> **Scenario:** Read geolocation and copy formatted string to clipboard

```
NF_Employee_TagLocation($Employee)
  → JSA_Device_GetLocation() → $LatLng
  → Change: $Employee/LastKnownLocation = $LatLng
  → JSA_Browser_CopyToClipboard($LatLng)
  → Show "Location tagged and copied."
```
