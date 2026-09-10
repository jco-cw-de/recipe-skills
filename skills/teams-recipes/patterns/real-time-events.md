# The `new_event` Trigger (Real-Time Events)

Workato's own Workbot for Microsoft Teams documentation does not mention this trigger at all. It was discovered by browsing the `teams_bot` connector's own trigger picker in the Workato recipe editor (picker label: **"New real-time event"**, described as *"Triggers when selected event occurs in Microsoft Teams"*, marked Beta).

## It's one generic trigger, not a family

Unlike `bot_command` (one trigger per registered command) or `help_event` (one fixed shape), `new_event` is a **single polymorphic trigger**. Its `event_name` input selects which specific real-time event it receives, and the trigger's *output schema changes depending on which event is selected* — this is a dynamic-schema (`dynamicPickListSelection`) trigger, not a static one.

```json
{
  "number": 0,
  "provider": "teams_bot",
  "name": "new_event",
  "as": "some-trigger-001",
  "keyword": "trigger",
  "dynamicPickListSelection": {
    "event_name": "Typeahead search (application/search)"
  },
  "input": {
    "event_name": "application/search"
  }
}
```

## `event_name` is a live-search field, not a static list

The recipe editor's "Event name" field is a typeahead search box (it has its own "Find" control), not a dropdown that shows every option up front. In this workspace, searching it for `tab`, `task`, `card`, `fetch`, `adaptive`, and `compose` all returned **no matches** — the only event that ever surfaced was:

- **Typeahead search** — `event_name` value: `application/search`

## What this means for Tab opened / Show tab using Adaptive Cards

Workato's documentation describes a separate "Tab opened trigger" and a "Show tab using Adaptive Cards" action. Given `new_event`'s polymorphic design, "Tab opened" is very likely just another `event_name` option on this same trigger (probably something like a `tab/fetch` Bot Framework invoke event) — not a distinct trigger the way the docs present it. Since no tab- or task-module-related event name surfaced when searched in this workspace, **it appears these are not available on this bot at all**, most likely because Workato's docs gate tab/file features behind an "Enterprise Workbot" tier/manifest that this connection isn't provisioned with. This is a checked, live finding (search returned zero matches across six plausible terms), not merely an untested gap — but it's still workspace-specific: a different `teams_bot` connection provisioned as an Enterprise Workbot might expose more `event_name` options. Re-check the picker on any new connection before assuming this holds universally.

## The `application/search` (Typeahead search) event's real schema

Confirmed live in the recipe editor (not activation-fired, but the editor resolved and returned this schema directly from the connector):

```json
{
  "name": "id",
  "channelId": "...",
  "serviceUrl": "...",
  "from": {
    "id": "...",
    "name": "...",
    "aadObjectId": "..."
  },
  "conversation": {
    "isGroup": false,
    "conversationType": "...",
    "tenantId": "...",
    "id": "..."
  },
  "recipient": {
    "id": "...",
    "name": "..."
  },
  "entities": [
    { "locale": "...", "country": "...", "platform": "...", "timezone": "...", "type": "..." }
  ],
  "channelData": {
    "channel": { "id": "..." },
    "team": { "id": "..." },
    "tenant": { "id": "..." },
    "source": { "name": "..." },
    "legacy": { "replyToId": "..." }
  },
  "replyToId": "...",
  "value": {
    "queryText": "...",
    "queryOptions": { "skip": 0, "top": 0 },
    "dataset": "..."
  },
  "locale": "...",
  "localTimezone": "...",
  "timestamp": "...",
  "localTimestamp": "..."
}
```

**Important structural difference from `bot_command`/`help_event`:** the caller-identity field is `from.aadObjectId` at the **top level** of the output, not nested under a `context` object (`context.from.aadObjectId`). Do not reuse the `bot_command`/`help_event` datapill path convention here without adjusting for this.

`value.queryText`/`value.queryOptions`/`value.dataset` is the actual search request — this event exists to let a recipe populate a dynamic, server-side-searched dropdown (an Adaptive Card `Input.ChoiceSet` with a `dynamic` style) as the user types.

## The paired `invoke_response` action

The picker's **"Response to real-time event"** action — internal name **`invoke_response`** — is presumably how a recipe returns results back to Teams for a `new_event` firing (e.g. the Typeahead search's result list). Confirmed real via direct picker selection, added as a step under a `new_event` (Typeahead search) trigger recipe. Its input was captured empty — no response-payload shape (e.g. how search results should be structured) has been configured or tested yet. A natural next step, not yet done: configure and activation-test a full `new_event` (Typeahead search) → `invoke_response` round trip against a live Adaptive Card with a dynamic `Input.ChoiceSet`.

## Verification status

`new_event` and `invoke_response` as names, and the exact output schema above for the `application/search` event specifically, were all confirmed live via the recipe editor's connector picker. The *absence* of tab/task-related events was confirmed by live search returning no matches for six plausible terms. Neither `invoke_response`'s response-payload shape, nor a full end-to-end typeahead-search recipe, has been built or activation-tested yet.
