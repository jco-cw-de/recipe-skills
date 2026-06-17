---
name: google-calendar-recipes
description: Google Calendar integration recipes for Workato. Enables AI agents to generate valid recipe JSON for Google Calendar operations including event management, availability checking, and calendar triggers.
license: MIT
metadata:
  author: Workato
  version: "1.0.0"
---

# Google Calendar Recipes Skill - Agent Instructions

> **DEPENDENCY: Load `/workato-recipes` first if not already loaded.**
> This skill requires the base Workato knowledge for triggers, control flow, datapills, formulas, and recipe structure.

This skill provides Google Calendar-specific knowledge for generating Workato recipes. It covers:

- **Native Google Calendar connector** (`provider: "google_calendar"`) — 8 native actions, 4 triggers
- **Adhoc HTTP** for Calendar API v3 endpoints not covered by native actions (free/busy, calendar list, etc.)

---

## CRITICAL: Pre-Generation Checklist

### For EXISTING projects:

1. **Read existing Google Calendar `.recipe.json` files** to understand local patterns
2. **Check the config section** for the connection name already in use

### For GREENFIELD projects:

1. **Use skill templates** — see `templates/create-event.json`, `templates/get-freebusy.json`
2. **Use descriptive UUIDs** — e.g., `create-event-001`, `fetch-freebusy-002`

### ALWAYS:

1. **Ask for connection name** — exact name of the Google Calendar connection in Workato
2. **Use calendar ID `"primary"`** unless the user specifies a different calendar
3. **Declare `extended_output_schema`** on every `__adhoc_http_action` step — the `output` field alone does not register datapill sources

---

## Table of Contents

1. [When to Use This Skill](#when-to-use-this-skill)
2. [Google Calendar Config Requirements](#google-calendar-config-requirements)
3. [Authentication](#authentication)
4. [Native Connector Guidance](#native-connector-guidance)
5. [Datapill Paths](#datapill-paths)
6. [Adhoc HTTP for Extended API Coverage](#adhoc-http-for-extended-api-coverage)
7. [Common Patterns](#common-patterns)
8. [Known Limitations and Gotchas](#known-limitations-and-gotchas)
9. [Validation](#validation)
10. [Templates](#templates)
11. [References](#references)

---

## When to Use This Skill

Use this skill when building Workato recipes that:

- Create, update, or delete Google Calendar events
- Search for events in a calendar by date or keyword
- Check availability / free-busy slots for scheduling
- React to calendar changes (new events, event start/end)
- Manage event attendees

**Prerequisites:**

- `workato-recipes` base skill loaded
- Workato workspace with Google Calendar connection configured (OAuth 2.0 or service account)
- Calendar ID (defaults to `"primary"` for the connected account's main calendar)

---

## Google Calendar Config Requirements

Every recipe using Google Calendar actions requires the `google_calendar` provider in the config section:

```json
{
  "keyword": "application",
  "provider": "google_calendar",
  "skip_validation": false,
  "account_id": {
    "zip_name": "my_google_calendar.connection.json",
    "name": "My Google Calendar",
    "folder": ""
  }
}
```

**Combined with API endpoint trigger:**

```json
"config": [
  { "keyword": "application", "provider": "workato_api_platform", "skip_validation": false, "account_id": null },
  { "keyword": "application", "provider": "google_calendar", "skip_validation": false, "account_id": { "zip_name": "my_google_calendar.connection.json", "name": "My Google Calendar", "folder": "" } }
]
```

---

## Authentication

The Google Calendar connector supports two auth methods:

**OAuth 2.0 (recommended for user-specific calendars):**

- Connect via Workato UI: New Connection → Google Calendar → OAuth 2.0 → Sign in with Google
- No manual scope configuration needed — Workato handles the OAuth flow

**Service account (recommended for shared/resource calendars):**

- Requires a GCP project, service account with Calendar API enabled
- Scopes needed: `calendar`, `calendar.events`, `admin.directory.resource.calendar`, `tasks`, `userinfo.email`
- See [Workato Google Calendar docs](https://docs.workato.com/connectors/google-calendar.html) for setup steps

---

## Native Connector Guidance

The Google Calendar connector provides 8 native actions and 4 triggers. See `lint-rules.json` for the authoritative list of valid action and trigger names.

### Choosing the Right Action

| Action                        | Use when                                                                   |
| ----------------------------- | -------------------------------------------------------------------------- |
| `create_event`                | Creating a timed event with start/end datetime                             |
| `create_all_day_event`        | Creating an event that spans a full day (no time)                          |
| `update_event`                | Modifying an existing event by its ID                                      |
| `delete_event`                | Removing an event from a calendar                                          |
| `get_event_by_id`             | Fetching full event details when you have the event ID                     |
| `search_events`               | Finding events by date range and/or search terms (returns a batch/list)    |
| `add_attendees_to_event`      | Adding participants to an existing event (batch)                           |
| `delete_attendees_from_event` | Removing participants from an existing event (batch)                       |
| `__adhoc_http_action`         | Any operation not listed above — free/busy, calendar list, quick add, etc. |

### Triggers

| Trigger             | Use when                                                      |
| ------------------- | ------------------------------------------------------------- |
| `new_event`         | React when a new event is created in a calendar               |
| `new_updated_event` | React when an event is created or updated                     |
| `event_start`       | Trigger at the start time of events (with optional lead time) |
| `event_end`         | Trigger at the end time of events                             |

All triggers require a **Calendar** input. Use `"primary"` for the connected account's default calendar.

---

## Datapill Paths

**Native actions** return data without a `["body"]` wrapper. Access fields directly:

```json
"path": ["id"]
"path": ["summary"]
"path": ["start", "date_time"]
"path": ["attendees"]
```

**`__adhoc_http_action`** — datapill paths match the fields declared in `extended_output_schema`.
The `output` string field defines internal parsing but does NOT register datapill sources.
Always declare `extended_output_schema` explicitly.

---

## Adhoc HTTP for Extended API Coverage

Use `__adhoc_http_action` for Calendar API v3 endpoints not covered by native actions.

The Google Calendar connector's base URL is `https://www.googleapis.com/calendar/v1/` —
use paths **without** a leading `/`:

```json
{
  "provider": "google_calendar",
  "name": "__adhoc_http_action",
  "input": {
    "verb": "post",
    "response_type": "json",
    "path": "calendar/v3/freeBusy",
    ...
  }
}
```

> **CRITICAL — `extended_output_schema` required:** The `output` string in
> `__adhoc_http_action` does not register the step as a datapill source.
> You must declare `extended_output_schema` for every field you reference downstream.

### Free/Busy (most common adhoc use case)

The freeBusy endpoint returns busy slots for one or more calendars. The `calendars`
field in the response is a **dynamic object keyed by calendar ID** — you cannot declare
its sub-fields statically.

**Recommended pattern:** Use `response_type: "raw"` to capture the entire response body
as a string, then parse it client-side or in a downstream step:

```json
{
  "number": 1,
  "provider": "google_calendar",
  "name": "__adhoc_http_action",
  "as": "fetch_freebusy",
  "keyword": "action",
  "uuid": "fetch-freebusy-001",
  "input": {
    "mnemonic": "Get free/busy",
    "verb": "post",
    "request_type": "json",
    "response_type": "raw",
    "path": "calendar/v3/freeBusy",
    "input": {
      "schema": "[{\"name\":\"timeMin\",\"type\":\"string\",\"label\":\"Time Min\",\"optional\":false},{\"name\":\"timeMax\",\"type\":\"string\",\"label\":\"Time Max\",\"optional\":false},{\"name\":\"timeZone\",\"type\":\"string\",\"label\":\"Timezone\",\"optional\":true},{\"name\":\"items\",\"type\":\"array\",\"label\":\"Items\",\"optional\":false,\"of\":\"object\"}]",
      "data": {
        "timeMin": "=_dp('{\"pill_type\":\"output\",\"provider\":\"workato_variable\",\"line\":\"vars\",\"path\":[\"target_date\"]}') + 'T00:00:00Z'",
        "timeMax": "=_dp('{\"pill_type\":\"output\",\"provider\":\"workato_variable\",\"line\":\"vars\",\"path\":[\"target_date\"]}') + 'T23:59:59Z'",
        "timeZone": "#{_dp('{\"pill_type\":\"output\",\"provider\":\"workato_variable\",\"line\":\"vars\",\"path\":[\"timezone\"]}')}",
        "items": "[{\"id\":\"primary\"}]"
      }
    },
    "output": "[{\"name\":\"body\",\"type\":\"string\",\"label\":\"Response body\"}]"
  },
  "extended_output_schema": [
    {
      "name": "body",
      "type": "string",
      "label": "Response body",
      "control_type": "text",
      "optional": true
    }
  ],
  "extended_input_schema": [
    {
      "name": "mnemonic",
      "label": "Mnemonic",
      "type": "string",
      "control_type": "text"
    },
    {
      "name": "verb",
      "label": "HTTP verb",
      "type": "string",
      "control_type": "text"
    },
    {
      "name": "request_type",
      "label": "Request type",
      "type": "string",
      "control_type": "text"
    },
    {
      "name": "response_type",
      "label": "Response type",
      "type": "string",
      "control_type": "text"
    },
    {
      "name": "path",
      "label": "Path",
      "type": "string",
      "control_type": "text"
    },
    {
      "name": "input",
      "label": "Request body",
      "type": "object",
      "properties": [
        {
          "name": "schema",
          "label": "Schema",
          "type": "string",
          "control_type": "text"
        },
        {
          "name": "data",
          "label": "Data",
          "type": "object",
          "properties": [
            {
              "name": "timeMin",
              "label": "Time Min",
              "type": "string",
              "control_type": "text"
            },
            {
              "name": "timeMax",
              "label": "Time Max",
              "type": "string",
              "control_type": "text"
            },
            {
              "name": "timeZone",
              "label": "Timezone",
              "type": "string",
              "control_type": "text"
            },
            {
              "name": "items",
              "label": "Items",
              "type": "string",
              "control_type": "text"
            }
          ]
        }
      ]
    },
    {
      "name": "output",
      "label": "Output schema",
      "type": "string",
      "control_type": "text"
    }
  ]
}
```

**Accessing the response body:**

```json
"path": ["body"]
```

The `body` field contains the raw JSON string. Parse it in a `py_eval` step or client-side.

**FreeBusy response structure:**

```json
{
  "kind": "calendar#freeBusy",
  "timeMin": "2026-06-18T00:00:00.000Z",
  "timeMax": "2026-06-18T23:59:59.000Z",
  "calendars": {
    "primary": {
      "busy": [
        { "start": "2026-06-18T09:00:00Z", "end": "2026-06-18T10:00:00Z" },
        { "start": "2026-06-18T14:00:00Z", "end": "2026-06-18T15:00:00Z" }
      ]
    }
  }
}
```

### List Calendars

```json
{
  "input": {
    "verb": "get",
    "response_type": "json",
    "path": "calendar/v3/users/me/calendarList"
  }
}
```

Declare `extended_output_schema` with a `items` array field to access the calendar list.

---

## Common Patterns

### 1. Create event from trigger data

```
Trigger: API endpoint (receive scheduling request)
Step 1: create_event (title, start, end, attendees from request)
Step 2: return_response 201 (return event ID)
```

### 2. Check availability before scheduling

```
Step 1: declare_variable — store target_date, timezone
Step 2: update_variables — populate from trigger
Step 3: __adhoc_http_action POST calendar/v3/freeBusy (response_type: raw)
Step 4: return_response — include body datapill as busy_blocks string
```

The caller (MCP App UI, agent, etc.) parses the raw freeBusy JSON to find open slots.

### 3. React to new calendar events

```
Trigger: new_event (calendar: "primary")
Step 1: [check conditions — event type, attendees, etc.]
Step 2: [downstream action — Slack notification, Salesforce update, etc.]
```

### 4. Event-based scheduling

```
Trigger: event_start (calendar: "primary", lead time: 15 minutes)
Step 1: get_event_by_id (confirm event details)
Step 2: [send reminder, update CRM, etc.]
```

---

## Known Limitations and Gotchas

### 1. `calendars` field is a dynamic object

The `calendars` field in freeBusy responses is keyed by calendar ID (e.g. `"primary"`,
`"user@example.com"`). You cannot declare this structure statically in
`extended_output_schema`. Options:

- Use `response_type: "raw"` — returns full body as string at `["body"]` — **recommended**
- Declare `"primary"` as a static sub-key — only works when calendar ID is always `"primary"`
- Use `response_type: "json"` with empty-properties object schema — datapill resolves to null

### 2. `request_type: "json"` required for POST requests

When using `__adhoc_http_action` for POST endpoints (like freeBusy), you must explicitly
set `request_type: "json"`. Without it the request body is not sent and Google Calendar
returns an empty response.

### 3. Path format — no leading slash

The Google Calendar connector's base URL includes a path prefix. Use paths **without**
a leading `/`:

```
CORRECT: "path": "calendar/v3/freeBusy"
WRONG:   "path": "/calendar/v3/freeBusy"
```

This is the opposite of the Asana connector which requires a leading `/`.

### 4. `extended_output_schema` always required for adhoc HTTP

The `output` string field in `__adhoc_http_action` defines internal parsing but does not
register the step as a datapill source. Without `extended_output_schema`, downstream steps
show "invalid source" in the Workato UI even though lint passes clean.

### 5. Native action datapills — no body wrapper

Native actions (`create_event`, `search_events`, etc.) return data directly without a
`["body"]` wrapper. Use `["id"]`, `["summary"]`, `["start", "date_time"]` etc. directly.

---

## Validation

See [validation-checklist.md](validation-checklist.md).

---

## Templates

See `templates/` directory:

- `create-event.json` — Create a calendar event with attendees via API endpoint
- `get-freebusy.json` — Fetch free/busy slots for a calendar using raw response mode

---

## References

- **Base Skill:** `workato-recipes` — Recipe structure, triggers, control flow, formulas
- **Google Calendar API v3:** https://developers.google.com/calendar/api/v3/reference
- **Workato Google Calendar docs:** https://docs.workato.com/connectors/google-calendar.html
- **Connector actions:** See `lint-rules.json` for the authoritative list of valid action/trigger names
