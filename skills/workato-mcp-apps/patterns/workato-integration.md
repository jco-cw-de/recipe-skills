# Workato Integration Patterns

How to connect an MCP App to a Workato recipe and MCP server.

---

## Recipe Requirements

A recipe that backs an MCP App must:

1. Use an **API endpoint trigger** (`workato_api_platform` / `receive_request`)
2. Return a `return_response` step with `http_status_code: "200"`
3. Serialize arrays and objects as JSON strings using `.to_json`
4. Be **running** (active) before the App can receive data

### Why .to_json for arrays/objects

`return_response` cannot map array or object datapills to array/object-typed response
fields — Workato silently strips them. Type response fields as `string` and serialize:

**Recipe:**

```json
{
  "input": {
    "http_status_code": "200",
    "response": {
      "items": "=_dp('{...}').to_json",
      "metadata": "=_dp('{...}').to_json"
    }
  }
}
```

**App:**

```javascript
const outer = JSON.parse(textItem.text);
const resp = outer.response || outer;
const items = typeof resp.items === "string" ? JSON.parse(resp.items) : [];
```

---

## MCP Server: Project Assets Mode

MCP Apps **require** project assets mode (`asset_type: project_assets`).
API collection mode does NOT support MCP Apps.

### Creating via UI (required — CLI doesn't support project assets mode)

1. AI Hub → MCP Servers → **+ Create an MCP server**
2. Choose **Project assets**
3. Select the project containing your recipe
4. Select the recipe(s) to expose as tools
5. Name the server, save

### Post-creation via CLI

```bash
# Add description
wk mcp servers update <handle> \
  --description "What this server does" \
  --store-type file --profile <name>

# Verify tool is active
wk mcp servers tools <handle> --store-type file --profile <name>
# Expect: active: true, enabled: true

# Test connectivity
wk mcp test "<mcp-url>?wkt_token=<token>" --store-type file --profile <name>
```

### Token management

The MCP server URL embeds the auth token as a query parameter:

```
https://XXXX.apim.mcp.trial.workato.com?wkt_token=<token>
```

Rotate the token when needed:

```bash
wk mcp servers token-renew <handle> --store-type file --profile <name>
```

---

## Linking the App to a Tool

In Workato UI:

1. AI Hub → MCP Servers → [your server] → **Apps** tab
2. **+ Add App**
3. Enter a name
4. **Linked tool** — select the tool name (recipe name with spaces as `_`)
5. **Start from scratch**
6. Paste your HTML into the code editor
7. **Save**

> The tool name shown in the Linked tool dropdown matches the recipe name
> as it appears in `wk mcp servers tools` output.

---

## Content Security Policy

Workato's default CSP for MCP Apps allows:

- `cdn.jsdelivr.net` — for Tailwind and ext-apps CDN
- `self` — for inline scripts

To allow additional external resources, configure in the App editor:
**Apps tab → [app name] → Content security policy**

Add domains to:

- **Resource domains** — for scripts, stylesheets, images, fonts
- **Connect domains** — for fetch/XHR/WebSocket calls
- **Frame domains** — for nested iframes

```
# Example: allow a charting library and its API
Resource domains: https://cdn.example.com
Connect domains: https://api.example.com
```

---

## Tool Name Conventions

Claude and other MCP clients use the tool name to decide when to call it.
Write your recipe's name and description to guide the model:

| Goal                             | Recipe name                            | Description                                   |
| -------------------------------- | -------------------------------------- | --------------------------------------------- |
| Always called with zero friction | Clear action verb + no required params | "No configuration needed — just call it."     |
| Called on specific request       | Descriptive noun phrase                | "Returns X for Y when the user asks about Z." |

**Remove parameters Claude should not supply.** Claude infers values from its own
integrations — if your recipe has a `workspace_id` param, Claude will try GIDs from
its own connected accounts. Hardcode these in the recipe and remove from the schema.

---

## Claude Desktop Setup

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "/path/to/npx",
      "args": [
        "mcp-remote",
        "https://XXXX.apim.mcp.trial.workato.com?wkt_token=<token>"
      ]
    }
  }
}
```

Find your npx path:

```bash
which npx
```

**Do not use `"type": "sse"`** — Claude Desktop silently skips entries in that format
with "not valid MCP server configuration". The `mcp-remote` bridge is required.

Quit and relaunch Claude Desktop after any config change.

> **Note on MCP App rendering in Claude Desktop:** Claude Desktop may render the App
> inline or fall back to text depending on the version and whether `ext-apps` support
> is enabled. The tool always works as a text-returning tool regardless.
