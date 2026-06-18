# Google Calendar API Reference

Reference for `__adhoc_http_action` endpoints on the `google_calendar` connector.

The connector's base URL is `https://www.googleapis.com/calendar/v1/`.
Use paths **without** a leading `/`.

---

## Choosing Between Native Actions and Adhoc HTTP

| Scenario                                               | Recommendation                                                        |
| ------------------------------------------------------ | --------------------------------------------------------------------- |
| Calendar ID known at recipe-write time (literal email) | `__adhoc_http_action` GET                                             |
| Calendar ID from user input / dynamic                  | Native `search_events` (user selects from picklist in UI)             |
| Creating events                                        | Native `create_event` — `calendar` field accepts a picklist selection |
| Listing events programmatically                        | `__adhoc_http_action` GET `/calendar/v3/calendars/{id}/events`        |

**Native actions with picklist fields (Calendar, Attendees, etc.) cannot be pre-populated from recipe JSON.** The field appears empty in the UI and blocks activation. Use adhoc HTTP when the value must come from code rather than a user-interactive picklist.

---

## Path Format

```
CORRECT: "path": "calendar/v3/freeBusy"
WRONG:   "path": "/calendar/v3/freeBusy"
```

---

## Key Endpoints

### Free/Busy

```
POST calendar/v3/freeBusy
```

Returns busy time intervals for a set of calendars.

**Request body:**

```json
{
  "timeMin": "2026-06-18T00:00:00Z",
  "timeMax": "2026-06-18T23:59:59Z",
  "timeZone": "America/Los_Angeles",
  "items": [{ "id": "primary" }]
}
```

**Response structure:**

```json
{
  "kind": "calendar#freeBusy",
  "timeMin": "...",
  "timeMax": "...",
  "calendars": {
    "<calendarId>": {
      "busy": [
        { "start": "2026-06-18T09:00:00Z", "end": "2026-06-18T10:00:00Z" }
      ]
    }
  }
}
```

> **IMPORTANT:** The `calendars` key is dynamic (keyed by calendar ID). Use
> `response_type: "raw"` to capture the full response as a string. Do NOT use
> `response_type: "json"` and try to declare `calendars` sub-fields — they will
> resolve to null because the key is dynamic.
>
> Always set `request_type: "json"` on POST requests or the body won't be sent.

**Recommended output schema (raw mode):**

```json
"extended_output_schema": [
  {"name": "body", "type": "string", "label": "Response body", "control_type": "text", "optional": true}
]
```

---

### List Calendars

```
GET calendar/v3/users/me/calendarList
```

Returns all calendars the user has access to.

**Response structure:**

```json
{
  "kind": "calendar#calendarList",
  "items": [
    {
      "id": "primary",
      "summary": "Chris Miller",
      "timeZone": "America/Los_Angeles",
      "accessRole": "owner",
      "primary": true
    }
  ]
}
```

---

### Get Calendar Metadata

```
GET calendar/v3/calendars/{calendarId}
```

Use `primary` for the connected account's main calendar.

---

### Quick Add Event

```
POST calendar/v3/calendars/{calendarId}/events/quickAdd?text=<event+text>
```

Creates an event from a natural language string. Pass the event description as a
query parameter `text`. Returns a full event object.

---

### Get Events List

```
GET calendar/v3/calendars/{calendarId}/events
```

Query parameters:

- `timeMin` — lower bound for event start time (ISO 8601)
- `timeMax` — upper bound for event end time (ISO 8601)
- `q` — free text search
- `singleEvents` — expand recurring events (`true`/`false`)
- `orderBy` — `startTime` or `updated`
- `maxResults` — max number of events to return (default 250)

---

## Extended Output Schema Requirement

For ALL `__adhoc_http_action` steps on `google_calendar`, you must declare
`extended_output_schema` explicitly. The `output` string field defines internal
parsing but does not register the step as a datapill source:

```json
"extended_output_schema": [
  {"name": "body", "type": "string", "label": "Response body", "control_type": "text", "optional": true}
]
```

Then reference downstream as:

```json
"path": ["body"]
```
