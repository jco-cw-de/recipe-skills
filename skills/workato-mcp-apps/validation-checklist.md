# MCP App Validation Checklist

Use this checklist before saving an MCP App in Workato.

---

## HTML Structure

- [ ] `<!DOCTYPE html>` present
- [ ] `<script type="module">` — ES module syntax required for CDN imports
- [ ] Tailwind CDN included: `<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>`
- [ ] `App` imported from exact URL: `https://cdn.jsdelivr.net/npm/@modelcontextprotocol/ext-apps@1.3.1/dist/src/app-with-deps.js`
- [ ] No other CDN scripts that are not listed in the Content Security Policy

## App Lifecycle

- [ ] `await app.connect()` called before any `on*` handlers are set
- [ ] `app.ontoolresult` set after `await app.connect()`
- [ ] Loading state shown by default (before tool result arrives)
- [ ] Error state handled — `try/catch` around `ontoolresult` body
- [ ] Loading state hidden when app content is shown
- [ ] Loading state hidden when error state is shown

## Tool Result Parsing

- [ ] Both result shapes handled: `result?.content` and `result?.result?.content`
- [ ] `content.find(c => c.type === 'text')` used to locate text item
- [ ] Missing text item handled gracefully (shows error, does not throw)
- [ ] For Workato recipes: outer wrapper parsed — `outer.response || outer`
- [ ] Array/object fields parsed from JSON strings: `typeof x === 'string' ? JSON.parse(x) : x`

## Sending Results

- [ ] `app.sendResult` guarded: `if (app && typeof app.sendResult === 'function')`
- [ ] Result payload is a plain object (not a DOM element or circular reference)
- [ ] User receives confirmation feedback after submission

## Event Handlers

- [ ] No inline `onclick=""` attributes — use `addEventListener` (CSP blocks inline handlers)
- [ ] Event listeners added after DOM elements exist (either in `render()` or after `innerHTML` assignment)

## Workato Server

- [ ] MCP server uses **project assets** mode (`asset_type: project_assets`) — NOT API collection mode
- [ ] Linked recipe is **running** (`active: true`) before testing the App
- [ ] Tool name in "Linked tool" dropdown matches the tool name in `wk mcp servers tools` output
- [ ] Server token is current (rotate with `wk mcp servers token-renew` if needed)

## Recipe Compatibility

- [ ] Recipe uses API endpoint trigger (`workato_api_platform` / `receive_request`)
- [ ] Recipe returns `return_response` with `http_status_code: "200"`
- [ ] Arrays and objects in response are serialized as strings with `.to_json`
- [ ] Response field types in recipe EIS match what the App expects to parse

## Content Security Policy

- [ ] All external CDN scripts are in CSP resource domains (Tailwind and ext-apps CDN are pre-allowed)
- [ ] All external API calls are in CSP connect domains
- [ ] No hardcoded external URLs in fetch/XHR that aren't in CSP connect domains

## Claude Desktop (if applicable)

- [ ] Config uses `mcp-remote` pattern — NOT `"type": "sse"`
- [ ] `npx` path is the absolute path (`which npx`)
- [ ] Token embedded as query param: `?wkt_token=<token>`
- [ ] Claude Desktop relaunched after config change
