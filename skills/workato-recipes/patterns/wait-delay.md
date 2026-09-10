# Pausing a Recipe Mid-Job (Wait / Delay)

To pause a recipe's execution for a fixed length of time before continuing to the next action — for example, waiting a few seconds before deleting a message that was just posted, or spacing out a sequence of API calls — use the `clock` connector's `wait_for_interval` action. This is a general-purpose platform action, not tied to any specific connector.

**UI label:** "Wait for specified length of time." Its own description: *"The job pauses for the specified length of time before carrying out subsequent actions in the recipe. Maximum time a job can be configured to remain paused is 732 days."*

## Shape

```json
{
  "number": 3,
  "provider": "clock",
  "name": "wait_for_interval",
  "as": "wait-004",
  "keyword": "action",
  "toggleCfg": {
    "interval": false
  },
  "input": {
    "interval": "3"
  },
  "uuid": "wait-004"
}
```

- `interval` is always a **number of seconds**, as a string — confirmed via the field's own UI label, "Interval (seconds)."
- `toggleCfg.interval` selects the input mode, same as any other toggleable field in this platform:
  - `false` — a custom value (a literal string, or a formula/datapill) in seconds, e.g. `"3"`.
  - `true` — a preset picklist (e.g. "5 minutes"). The stored `input.interval` value is still the equivalent number of seconds as a string, e.g. `"300"` for 5 minutes — the picklist is purely a UI convenience, the underlying field is identical either way.

Use `false`/a raw literal for short, precise waits (a handful of seconds); either mode works for longer, round-number waits.

## Requires the `clock` provider in config

Same as the [scheduler trigger](../triggers/scheduler.md) — `account_id: null`, no real connection needed:

```json
{
  "keyword": "application",
  "name": "clock",
  "provider": "clock",
  "skip_validation": false,
  "account_id": null
}
```

## Verification

Confirmed by a human tester adding this action live in the Workato recipe editor (not merely guessed), for both the custom-seconds mode and the preset-picklist mode, then re-pulling the recipe and reading its actual saved JSON for each. No production recipe in the source workspace was found using this action prior to this — it was previously undocumented anywhere in this repo.

## Validation Checklist

### `wait_for_interval`
- [ ] `provider` is `clock`, not a connector-specific provider — this action is available in any recipe, not tied to whatever connector the surrounding steps use
- [ ] `input.interval` is a string containing a plain number of **seconds** — do not add a separate unit field or assume minutes/hours
- [ ] Config includes a `clock` provider entry with `account_id: null` (no real connection needed, same as the scheduler trigger)
