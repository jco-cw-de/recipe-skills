# Google Calendar Validation Checklist

> **Run the base checklist first:** See [workato-recipes/validation-checklist.md](../workato-recipes/validation-checklist.md) for base recipe validation.

The following checks are specific to Google Calendar connector recipes.

---

## Config & Connection

- [ ] Config includes `google_calendar` provider with connection reference
- [ ] Connection `account_id` has correct `name` matching the workspace's Google Calendar connection
- [ ] If using adhoc HTTP, the `google_calendar` provider is still in config

## Actions

- [ ] Action `name` matches a valid name in `lint-rules.json` or is `__adhoc_http_action`
- [ ] `keyword` is `"action"` for all non-trigger steps

## Adhoc HTTP (`__adhoc_http_action`)

- [ ] Path does NOT use a leading `/` — e.g. `calendar/v3/freeBusy` not `/calendar/v3/freeBusy`
- [ ] `request_type: "json"` is set on all POST requests (required or body is not sent)
- [ ] `extended_output_schema` is declared for every field referenced by downstream datapills
- [ ] For freeBusy: use `response_type: "raw"` to capture the dynamic `calendars` object as a string

## Datapill Paths

- [ ] Native action datapill paths do NOT include `["body"]` wrapper — use `["id"]`, `["summary"]` etc. directly
- [ ] Adhoc HTTP datapill paths match fields declared in `extended_output_schema`

## Calendar ID

- [ ] Calendar ID defaults to `"primary"` unless a specific calendar is required
- [ ] Calendar ID is passed as a string, not hardcoded integer

## Triggers

- [ ] `new_event` / `new_updated_event` have a Calendar input specified
- [ ] `event_start` / `event_end` have Calendar and optional lead time configured

## Time Handling

- [ ] Datetime values use ISO 8601 format: `YYYY-MM-DDTHH:MM:SSZ`
- [ ] Timezone is passed as an IANA string (e.g. `America/Los_Angeles`, `UTC`)
- [ ] freeBusy `timeMin`/`timeMax` are full ISO 8601 strings including `T` and `Z`
