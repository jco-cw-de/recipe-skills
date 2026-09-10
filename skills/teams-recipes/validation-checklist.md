# Teams Validation Checklist

> **Run the base checklist first:** See [workato-recipes/validation-checklist.md](../workato-recipes/validation-checklist.md) for base recipe validation.

The following checks are specific to the Workbot for Microsoft Teams (`teams_bot`) connector.

---

## Config & Connection

- [ ] Config includes a `teams_bot` provider entry with a real connection `account_id`
- [ ] Action/trigger `name` matches a valid name in `lint-rules.json` or is `__adhoc_http_action`

## Datapill Paths

- [ ] `bot_command` trigger datapills do NOT use a `["body"]` wrapper (native trigger, not adhoc HTTP)
- [ ] Trigger parameters are read from `["parameters", "<param_name>"]`; caller context (e.g. the invoking user's Azure AD object ID) is read from `["context", "from", "aadObjectId"]`

## `bot_command` Trigger

- [ ] `action_name` is a unique command name across this Teams bot connection — do not reuse a command name already registered by another recipe
- [ ] `parameters` is a JSON-stringified array of parameter definitions (`name`, `label`, `type`, `control_type`, `optional`), matching `extended_output_schema`'s `parameters` object
- [ ] A date parameter uses `"type": "date_time"` with `"control_type": "date"` (not `"type": "date"`) — and its `extended_output_schema` entry includes `"parse_output": "date_conversion"` / `"render_input": "date_conversion"`
- [ ] `input.hide_from_help` is present as a string (`"true"` or `"false"`), matching production usage

## `help_event` Trigger

- [ ] `input` is empty (`{}`) — this trigger takes no input
- [ ] Only one `help_event` recipe is intended to be active per bot at a time — flag to the user if this looks like it would create a second one
- [ ] Caller context is read from `["context", "from", "aadObjectId"]`, same path as `bot_command` — do not assume a different structure without confirming first

## `new_message_event` Trigger

- [ ] Output field paths used (e.g. `message_text`, `conversation_id`) are flagged to the user as unconfirmed guesses, not asserted as verified — no live schema has been captured for this trigger yet

## `new_event` Trigger

- [ ] `input.event_name` is a real, live-search-confirmed value (currently only `application/search` is confirmed available) — never invent an `event_name` value from the doc's feature list (e.g. anything tab/task-related) without checking the picker first
- [ ] Caller identity is read from `["from", "aadObjectId"]` (top-level), NOT `["context", "from", "aadObjectId"]` — this trigger's shape differs from `bot_command`/`help_event`

## `get_user_by_principal_name` Action

- [ ] Input uses exactly one of `principal_name` (email/UPN) or `id` (Azure AD object ID) — do not include both

## `post_blocks_message` Action

- [ ] `channel` is set — this action always targets a specific channel/user, unlike `post_blocks_reply_message`
- [ ] `blocks` is an array where each entry has a `block_type` (`text_block` or `text_with_button_block`) and a matching nested object with that same key
- [ ] `text_block` entries use `"text_type": "body_text"` for standard message text, or `"text_type": "custom"` (with `"separator": "true"` and `"style": {"isSubtle": "true"}`) for a de-emphasized footer line — not a made-up `text_type` value
- [ ] `text_with_button_block` entries include `bot_command` (the command this button should invoke), `button_type` (observed value: `"submit"`), `open_task_module` (string `"true"`/`"false"`), and `params` (a JSON string of the parameters that command's `bot_command` trigger expects)
- [ ] When a `text_with_button_block`'s `params` value must be computed (not a straight datapill passthrough), the field is built in full formula mode (leading `=`), not as a template string with an embedded formula expression

## `post_blocks_reply_message` Action

- [ ] Do NOT set a `channel` field — this action has none; it replies to the invoking context implicitly
- [ ] Only used from a recipe with a real Workbot-command invocation context (`bot_command` or `help_event`), never from a recipe with no live Teams invocation (e.g. a clock-triggered recipe)
- [ ] Not `post_blocks_message` — verify the recipe actually needs "reply where invoked" semantics, not "post to a specific channel/user"

## `delete_message` Action

- [ ] `message_id` is sourced from a `post_blocks_message`/`post_blocks_reply_message` step's `id` output
- [ ] `conversation_id` is NOT sourced from `get_user_by_principal_name`'s `id` output (confirmed wrong via a live job failure — a Teams user ID is not a conversation ID) — flag to the user that the correct source for this field is still unconfirmed
