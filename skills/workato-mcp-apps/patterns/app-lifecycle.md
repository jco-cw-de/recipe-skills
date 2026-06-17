# MCP App Lifecycle

Every MCP App follows the same lifecycle: connect → receive → render → submit.
Understanding the order matters — missing `await app.connect()` or setting handlers
too late are the most common sources of silent failures.

---

## Full Lifecycle

```
1. Browser loads HTML in sandboxed iframe
2. Script executes: new App({name, version})
3. await app.connect() — sends initialize handshake to host, waits for response
4. app.ontoolresult = fn  — register handler (AFTER connect completes)
5. [Host calls tool → recipe executes → result returns]
6. fn(result) fires — render UI from data
7. User interacts
8. app.sendResult(data) — return structured output to host
9. [Host may call the tool again → step 5 repeats]
```

---

## 1. Instantiation

```javascript
const app = new App({ name: "My App", version: "1.0.0" });
```

`name` and `version` appear in the host's tool call metadata. Use descriptive names.

---

## 2. Connect

```javascript
await app.connect();
```

**This must complete before setting any handlers.** `connect()` performs the
initialization handshake with the host via PostMessage. The host will not deliver
tool results until the handshake is complete.

**Common mistake — setting handler before connect:**

```javascript
// WRONG — may miss the result
app.ontoolresult = fn;
await app.connect();

// CORRECT
await app.connect();
app.ontoolresult = fn;
```

---

## 3. Receiving Tool Results

```javascript
app.ontoolresult = (result) => {
  // result shape from Workato API endpoint recipes:
  // {
  //   content: [{ type: 'text', text: '{"http_status_code":"200","response":{...}}' }]
  // }

  const content = result?.content || result?.result?.content || [];
  const textItem = content.find((c) => c.type === "text");
  if (!textItem) {
    /* handle error */ return;
  }

  const data = JSON.parse(textItem.text);
  render(data);
};
```

### Result shape variants

Hosts may wrap the result differently. Defensively handle both:

```javascript
const content =
  result?.content || // direct content array
  result?.result?.content || // wrapped in result object
  [];
```

### Workato API endpoint recipe wrapper

Workato's `return_response` action wraps the payload:

```javascript
const textItem = content.find((c) => c.type === "text");
const outer = JSON.parse(textItem.text);
// outer = { "http_status_code": "200", "response": { ...your fields... } }
const data = outer.response || outer;
```

### Arrays and objects serialized as strings

Workato's `return_response` cannot map array or object datapills directly — they
get stripped. The recipe must serialize them with `.to_json`. Parse them in the app:

```javascript
// Recipe: "items": "=_dp(...).to_json"
// App:
const items =
  typeof data.items === "string" ? JSON.parse(data.items) : data.items || [];
```

---

## 4. Other notification handlers

```javascript
// Fires before tool execution — streaming partial arguments
app.ontoolinputpartial = (partial) => {
  /* show loading with context */
};

// Fires with complete tool arguments just before execution
app.ontoolinput = (input) => {
  /* update UI to show what's being fetched */
};

// Fires if the user or host cancels the tool call
app.ontoolcancelled = () => {
  /* reset UI to idle state */
};

// Fires when host context changes (theme, locale, etc.)
app.onhostcontextchanged = (ctx) => {
  /* adapt UI to new theme/locale */
};
```

---

## 5. Sending Results Back

```javascript
// Always guard — not all hosts implement sendResult
if (app && typeof app.sendResult === "function") {
  await app.sendResult({
    action: "accepted",
    items: selectedItems,
  });
}
```

`sendResult` delivers structured output to the host, which can feed back into the
conversation or trigger further tool calls. Use it for:

- Approval/rejection decisions
- Form submissions
- User selections that drive downstream actions

---

## 6. Loading and Error States

Always show a loading state while waiting for the tool result. The host renders the
App immediately — before the tool has finished executing.

```javascript
// Show loading immediately on mount (default HTML state)
// Hide and show app content in ontoolresult

app.ontoolresult = (result) => {
  try {
    const data = parseResult(result);
    render(data); // hides loading, shows app
  } catch (err) {
    showError(`Failed to load: ${err.message}`); // hides loading, shows error
  }
};
```

Recommended DOM pattern:

```html
<div id="loading"><!-- spinner --></div>
<div id="error" class="hidden"><!-- error message --></div>
<div id="app" class="hidden"><!-- main content --></div>
```

```javascript
function showError(msg) {
  document.getElementById("loading").classList.add("hidden");
  const el = document.getElementById("error");
  el.textContent = msg;
  el.classList.remove("hidden");
}

function showApp() {
  document.getElementById("loading").classList.add("hidden");
  document.getElementById("app").classList.remove("hidden");
}
```
