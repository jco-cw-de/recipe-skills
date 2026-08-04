---
name: teams-recipes
description: Microsoft Teams Workbot (teams_bot) recipes for Workato. Enables AI agents to generate valid recipe JSON for bot commands, user lookup, and block message posting.
license: MIT
metadata:
  author: Your Name
  version: "1.0.0"
---

# Teams Recipes Skill

> **DEPENDENCY: Load the `workato-recipes` base skill first if not already loaded.**
> This skill requires the base Workato knowledge for triggers, control flow, datapills, and recipe structure.

You are now equipped with knowledge for writing Workato recipes using **Workbot for Microsoft Teams** (`teams_bot`) — a connector for receiving slash-style bot commands from Teams users and posting block-based messages back.

---

## CRITICAL: Pre-Generation Checklist

### For EXISTING projects:
1. **Read existing recipes using the `teams_bot` provider** to see what bot commands are already registered on the target connection
2. **Never reuse an existing `action_name`** on the same connection — each `bot_command` trigger's `action_name` must be unique across the whole Teams bot

### For GREENFIELD projects:
1. **Use the skill template** — see `templates/bot-command-notify.json` for a validated example
2. **Use descriptive UUIDs** — e.g., `lookup-caller-002`, `post-notify-003`

### ALWAYS:
1. **Ask for the connection name** — exact name of the Teams Workbot connection in Workato
2. **Confirm the desired `action_name`** — this becomes the Teams-visible command name; check it isn't already used
3. **Decide the user-lookup mode** — by `principal_name` (email/UPN) or `id` (Azure AD object ID) — see [Native Connector Guidance](#native-connector-guidance)

---

## Table of Contents

1. [When to Use This Skill](#when-to-use-this-skill)
2. [Teams Config Requirements](#teams-config-requirements)
3. [Native Connector Guidance](#native-connector-guidance)
4. [Teams Datapill Paths](#teams-datapill-paths)
5. [Block Types Reference](#block-types-reference)
6. [Common Patterns](#common-patterns)
7. [Validation](#validation)
8. [Templates](#templates)
9. [References](#references)

---

## When to Use This Skill

Use this skill when a recipe needs to expose a command that Teams users invoke from within Microsoft Teams (via Workbot), look up the invoking (or another) Teams user's identity, and/or post a message — including one with an actionable button — into a Teams channel or DM. This is not a general Microsoft Graph/Teams API integration; it only covers what the `teams_bot` connector's own actions and trigger expose.

---

## Teams Config Requirements

```json
{
  "keyword": "application",
  "name": "teams_bot",
  "provider": "teams_bot",
  "skip_validation": false,
  "account_id": 558004
}
```

`account_id` must reference a real Teams Workbot connection — unlike the native Email connector, this one requires an actual connection.

---

## Native Connector Guidance

The Teams connector provides 2 verified native actions and 1 verified trigger. See `lint-rules.json` for the authoritative list.

### Choosing the Right Trigger

- **`bot_command`** — Use to expose a new slash-style command to Teams users. Requires a unique `action_name` and a `parameters` field (a JSON-stringified array of parameter definitions — each needs `name`, `label`, `type`, `control_type`, `optional`). The trigger's output includes both the declared `parameters` (per your schema) and a `context` object carrying details about the invoking user and conversation (e.g. `context.from.aadObjectId`).

### Choosing the Right Action

- **`get_user_by_principal_name`** — Looks up a Teams user. Despite the name, this action has been observed in live recipes with **two different input fields**: `principal_name` (an email/UPN string) in one recipe, and `id` (an Azure AD object ID — typically taken straight from a `bot_command` trigger's `context.from.aadObjectId`) in three others. Use `id` when you already have the caller's AAD object ID from a `bot_command` trigger context; use `principal_name` when you only have an email address (e.g. sourced from another system). Do not set both fields on the same action.
- **`post_blocks_message`** — Posts a message built from one or more `blocks` to a channel or user. Use this any time a bot command needs to reply. See [Block Types Reference](#block-types-reference) for what each block type supports.

---

## Teams Datapill Paths

Native action/trigger datapills do NOT use a `["body"]` wrapper. Notable paths:

- Trigger declared parameters: `["parameters", "<param_name>"]`
- Trigger caller context: `["context", "from", "aadObjectId"]`
- `get_user_by_principal_name` output: `["id"]` (the Teams user ID — feed this into `post_blocks_message`'s `channel` field to message that user directly)

---

## Block Types Reference

`post_blocks_message`'s `blocks` array has been observed with two block types in live recipes:

**`text_block`** — plain or markdown-ish text:
```json
{
  "block_type": "text_block",
  "text_block": {
    "text": "Some message text, markdown-flavored links/bold supported",
    "text_type": "body_text"
  }
}
```

**`text_with_button_block`** — text with an actionable button that invokes another `bot_command`:
```json
{
  "block_type": "text_with_button_block",
  "text_with_button_block": {
    "text": "Snooze this for 30 days?",
    "button_title": "Snooze",
    "button_type": "submit",
    "bot_command": "rm_snoozerp",
    "params": "{\"rpid\": \"12345\"}",
    "open_task_module": "false"
  }
}
```

`params` is a JSON-encoded string matching the target command's `bot_command` trigger's own `parameters` schema — build it with string concatenation/formula mode when values come from datapills (see the real usage pattern in `templates/bot-command-notify.json`'s sibling actions, or the base skill's [variables-and-lists.md](../workato-recipes/patterns/variables-and-lists.md) for formula-mode string building).

No other block types have been verified — do not assume additional block types (e.g. images, adaptive-card JSON blocks) exist without confirming first (see the verification note in `lint-rules.json`).

---

## Common Patterns

### Command → Lookup → Reply

The standard shape for a Teams bot command: receive the command, resolve the caller's Teams user record, then reply to them directly.

1. `bot_command` trigger declares the command's parameters
2. `get_user_by_principal_name` resolves the caller using `context.from.aadObjectId` from the trigger
3. `post_blocks_message` sends the reply, using the lookup action's `id` output as the `channel`

See `templates/bot-command-notify.json` for the full working example.

### Button That Invokes Another Command

A `text_with_button_block` can chain into a second bot command: set `bot_command` to that command's `action_name`, and `params` to a JSON string of the values that command's own trigger `parameters` expect.

---

## Validation

See [validation-checklist.md](validation-checklist.md) for the full checklist. Always run the base checklist first.

---

## Templates

- [`templates/bot-command-notify.json`](templates/bot-command-notify.json) — A `bot_command` trigger that looks up the calling user and posts a text-block reply. Structurally validated by import against a live Workato workspace; the two actions it uses (`get_user_by_principal_name`, `post_blocks_message`) were additionally activation-verified in a scratch recipe. The trigger itself was intentionally not activation-tested to avoid registering a live, Teams-visible bot command as a side effect — its shape is instead backed by two identical real production recipes.

---

## References

- **Base Skill:** `workato-recipes` — recipe structure, triggers, control flow, formulas
- **Related:** `slack-recipes`' `slack_bot` variant — the closest structural precedent (a chat-platform Workbot), useful for comparison if extending this skill further
