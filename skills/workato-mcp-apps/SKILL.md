---
name: workato-mcp-apps
description: Build MCP Apps — interactive HTML UIs that render inline in Claude, ChatGPT, and other MCP-compatible hosts. Covers the ext-apps SDK, App class lifecycle, tool result handling, Workato MCP server setup, and Claude Desktop integration.
license: MIT
metadata:
  author: Workato
  version: "1.0.0"
---

# MCP Apps Skill — Agent Instructions

> **STANDALONE SKILL — no base skill dependency required.**

An **MCP App** is an HTML page that renders inline in an LLM chat host (Claude, ChatGPT, VS Code, etc.) as a sandboxed iframe. It receives tool execution results and can send responses back to the host. Use it when a tool's output benefits from a visual, interactive UI — sortable tables, forms, dashboards, drag-and-drop interfaces.

---

## Table of Contents

1. [When to Use MCP Apps](#when-to-use-mcp-apps)
2. [Architecture](#architecture)
3. [Required HTML Structure](#required-html-structure)
4. [App Lifecycle](#app-lifecycle)
5. [Receiving Tool Results](#receiving-tool-results)
6. [Sending Results Back](#sending-results-back)
7. [Styling](#styling)
8. [Security and CSP](#security-and-csp)
9. [Workato Setup](#workato-setup)
10. [Claude Desktop Integration](#claude-desktop-integration)
11. [Known Gotchas](#known-gotchas)
12. [Templates](#templates)

---

## When to Use MCP Apps

Use an MCP App when:

- Tool output is a list, table, or structured dataset the user needs to filter, sort, or interact with
- The user needs to make selections or approvals before an action is taken (e.g. approve/reject rows)
- You need drag-and-drop, charts, or any interaction beyond text
- The tool returns data that maps to a form the user should fill out

Do NOT use MCP Apps for:

- Simple text or one-liner responses
- Tools that only need to show a confirmation

---

## Architecture

```
LLM calls tool
  └── Workato recipe runs → returns JSON text content
  └── Host renders MCP App HTML in sandboxed iframe
  └── App receives result via app.ontoolresult
  └── App renders UI
  └── User interacts → app.sendResult() → host receives structured output
```

The MCP App HTML is authored in Workato's AI Hub:
**AI Hub → MCP Servers → [server] → Apps tab → + Add App**

The linked tool must use **project assets mode** (`asset_type: project_assets`).
API collection mode does NOT support MCP Apps.

---

## Required HTML Structure

Every MCP App must follow this exact base structure:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>App</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
  </head>
  <body>
    <!-- Build your app UI here -->

    <script type="module">
      // Most cases, do not modify this line.
      import { App } from "https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js";

      (async () => {
        const app = new App({ name: "My App", version: "1.0.0" });
        await app.connect();

        // Application logic goes here
      })();
    </script>
  </body>
</html>
```

**Critical rules:**

- `<script type="module">` — required for ES module imports
- `App` import from `ext-apps@1.3.1` CDN — do not change this URL
- `await app.connect()` — must be called before setting any handlers
- Tailwind CDN for styling — already allowed by Workato's default CSP

---

## App Lifecycle

```
1. new App({name, version})   — instantiate
2. await app.connect()        — establish PostMessage transport with host
3. app.ontoolresult = fn      — register handler before tool fires
4. [host calls tool]          — recipe executes, result arrives
5. fn(result) fires           — render UI from result data
6. user interacts             — update UI state
7. app.sendResult(data)       — return structured output to host
```

**`app.connect()` must complete before any `on*` handler fires.** The host will not
deliver tool results until the app has completed its initialization handshake.

---

## Receiving Tool Results

Set `app.ontoolresult` after `await app.connect()`:

```javascript
app.ontoolresult = (result) => {
  // result.content is an array of content items
  const content = result?.content || result?.result?.content || [];
  const textItem = content.find((c) => c.type === "text");
  if (!textItem) {
    /* handle error */ return;
  }

  // Parse the tool's JSON output
  const data = JSON.parse(textItem.text);

  // Render your UI
  renderUI(data);
};
```

**Tool result shape from Workato API endpoint recipes:**

```javascript
// textItem.text contains the return_response body:
{
  "http_status_code": "200",
  "response": {
    "field1": "value",
    "array_field": "[...]",  // arrays serialized as JSON strings via .to_json
    "object_field": "{...}"  // objects serialized as JSON strings via .to_json
  }
}
```

**Parsing arrays/objects that were serialized as strings:**

```javascript
const outer = JSON.parse(textItem.text);
const resp = outer.response || outer;

// Arrays returned as .to_json strings — parse them
const items =
  typeof resp.items === "string" ? JSON.parse(resp.items) : resp.items || [];
```

**Why serialized as strings:** Workato's `return_response` cannot map array or object
datapills directly to array-typed response fields — they get stripped. The pattern is
to type response fields as `string` and use `=_dp(...).to_json` in the recipe. The App
parses them back client-side.

---

## Sending Results Back

Call `app.sendResult(data)` to return structured output to the host:

```javascript
$('btn-accept').addEventListener('click', async () => {
  const result = {
    accepted_items: [...],
    action: 'confirmed'
  };
  await app.sendResult(result);
  // Show confirmation in UI
});
```

The host receives this as the app's structured output. Use it for approval flows,
form submissions, or any user decision that should feed back into the conversation.

---

## Styling

Use **Tailwind CSS via CDN** (already loaded in the base template):

```html
<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
```

Tailwind CDN is allowed by Workato's default Content Security Policy for MCP Apps.
No additional CSP configuration needed for Tailwind.

**Common UI patterns:**

```html
<!-- Loading spinner -->
<div class="flex flex-col items-center justify-center min-h-48 gap-3">
  <div
    class="w-8 h-8 border-4 border-blue-200 border-t-blue-500 rounded-full animate-spin"
  ></div>
  <p class="text-gray-500 text-sm">Loading…</p>
</div>

<!-- Error state -->
<div
  id="error"
  class="hidden bg-red-50 border border-red-200 rounded-lg p-4 text-red-700 text-sm"
></div>

<!-- Card -->
<div class="bg-white rounded-xl shadow-sm border border-gray-100 p-4"></div>

<!-- Primary button -->
<button
  class="px-4 py-2 bg-blue-600 text-white text-sm font-medium rounded-lg hover:bg-blue-700 transition-colors"
>
  Submit
</button>
```

**Always include loading and error states.** The app starts loading before the tool
result arrives. Show a spinner immediately, then swap to content in `ontoolresult`.

---

## Security and CSP

MCP Apps run in a sandboxed iframe. By default Workato's CSP allows:

- `cdn.jsdelivr.net` — for Tailwind and ext-apps CDN imports
- `self` — for inline scripts

**To use additional external resources**, configure CSP in Workato's App editor:
AI Hub → MCP Servers → [server] → Apps → [app] → Content security policy

**No inline event handlers** — use `addEventListener`. Inline `onclick=""` attributes
may be blocked by CSP.

**No `localStorage` or `sessionStorage`** — apps are sandboxed with a unique origin
per session. Use in-memory state only.

---

## Workato Setup

### 1. Recipe requirements

The linked recipe must:

- Use an **API endpoint trigger** (`workato_api_platform` / `receive_request`)
- Return a `return_response` step with `http_status_code: "200"`
- Serialize complex response fields (arrays, objects) as JSON strings using `.to_json`
- Be **running** (active) before the MCP App can be linked

### 2. MCP server — project assets mode

```bash
# CLI only creates API collection mode — NOT compatible with MCP Apps
# Use the UI to create a project assets server:
# AI Hub → MCP Servers → + Create → Project assets → select recipe
```

After creation, add description via CLI:

```bash
wk mcp servers update <handle> --description "..." --store-type file --profile <name>
```

Verify:

```bash
wk mcp servers tools <handle> --store-type file --profile <name>
# Confirm active: true, enabled: true
```

### 3. Link the App to the tool

In Workato UI:

1. AI Hub → MCP Servers → [your server] → **Apps** tab
2. **+ Add App**
3. Name the app
4. **Linked tool** — select your recipe's tool name
5. **Start from scratch** — paste your HTML
6. **Save**

> The linked tool must match the tool name exposed by the MCP server. For Workato
> project assets servers, the tool name is the recipe name with spaces replaced by `_`.

---

## Claude Desktop Integration

Claude Desktop does **not** support `type: sse` config entries. Use `mcp-remote`:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "/path/to/npx",
      "args": ["mcp-remote", "https://<mcp-url>?wkt_token=<token>"]
    }
  }
}
```

Config file location: `~/Library/Application Support/Claude/claude_desktop_config.json`

**Quit and relaunch Claude Desktop** after any config change — it does not hot-reload.

> **Note:** Claude Desktop may or may not render the MCP App UI inline depending on
> version. The tool always works as a text-returning tool. The MCP App UI rendering
> requires a host that supports the `ext-apps` spec (Claude.ai, some VS Code versions).

### Tool schema design for Claude

Claude infers parameter values from its own integrations. To prevent Claude from
passing wrong values (e.g. workspace GIDs from its own accounts):

- **Remove parameters Claude should not supply** — hardcode them in the recipe
- Make the tool description explicit: `"No configuration needed — just call it."`
- Only expose parameters the user genuinely controls (date, timezone, preferences)

---

## Known Gotchas

### 1. Arrays serialized as strings

Workato's `return_response` strips array/object datapills. Serialize with `.to_json`
in the recipe and parse in the App:

```javascript
// Recipe: "tasks": "=_dp(...).to_json"
// App:
const tasks = typeof resp.tasks === "string" ? JSON.parse(resp.tasks) : [];
```

### 2. `await app.connect()` must complete first

Setting `ontoolresult` before `connect()` completes will miss the first event.
Always `await app.connect()` then set handlers.

### 3. `app.sendResult` may not exist on all hosts

Check before calling:

```javascript
if (app && typeof app.sendResult === "function") {
  await app.sendResult(result);
}
```

### 4. No `window.__mcp_tool_result__` in production

Some older documentation shows `window.__mcp_tool_result__` as the data source.
Use `app.ontoolresult` instead — it is the correct ext-apps SDK pattern.

### 5. CSP blocks inline event handlers

Do not use `onclick=""` attributes. Use `addEventListener` instead.

### 6. Linked tool must be running

The recipe must be in `running: true` state before the MCP App can deliver results.
Start the recipe from the Workato UI if `wk recipes start` times out.

---

## Templates

See `templates/` directory:

- `base.html` — Minimal base structure matching the Workato default template
- `data-table.html` — Sortable, filterable table with accept/reject actions

See `patterns/` directory:

- `patterns/app-lifecycle.md` — Full lifecycle reference with JS examples
- `patterns/workato-integration.md` — Recipe requirements, MCP server setup, Claude Desktop config

---

## References

- **ext-apps spec:** https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx
- **ext-apps SDK docs:** https://apps.extensions.modelcontextprotocol.io/api/classes/app.App.html
- **Workato MCP Apps docs:** https://docs.workato.com/en/mcp/mcp-apps.html
- **CDN:** `https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js`
