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

## `get_user_by_principal_name` Action

- [ ] Input uses exactly one of `principal_name` (email/UPN) or `id` (Azure AD object ID) — do not include both

## `post_blocks_message` Action

- [ ] `blocks` is an array where each entry has a `block_type` (`text_block` or `text_with_button_block`) and a matching nested object with that same key
- [ ] `text_block` entries use `"text_type": "body_text"` for standard message text, or `"text_type": "custom"` (with `"separator": "true"` and `"style": {"isSubtle": "true"}`) for a de-emphasized footer line — not a made-up `text_type` value
- [ ] `text_with_button_block` entries include `bot_command` (the command this button should invoke), `button_type` (observed value: `"submit"`), `open_task_module` (string `"true"`/`"false"`), and `params` (a JSON string of the parameters that command's `bot_command` trigger expects)
- [ ] When a `text_with_button_block`'s `params` value must be computed (not a straight datapill passthrough), the field is built in full formula mode (leading `=`), not as a template string with an embedded formula expression
