# Debugging Automations

When an automation didn't fire — or fired and produced something unexpected — the
fastest path to a root cause is the platformModel **events API**. The events
endpoint returns a structured audit log of every mutation, condition evaluation,
and action result for a record, which lets you replace UI guesswork with direct
inspection of what the engine actually did.

This guide leads with a **diagnostic playbook** so you can map a symptom to a
query and an interpretation, then drills into the endpoint and response shape.
For the catalog of action keys an automation may use, see `automations.md`. For
auth and base-URL conventions, see `api-platformmodel.md`.

## Diagnostic playbook

Always start by pulling the events for the affected record and scanning the
timeline. Almost every automation issue resolves into one of four shapes below.

### Symptom 1 — "My automation didn't run at all"

**Query:** fetch events for the record around the suspected trigger time and
look for the **trigger event itself** (a `mutate` for field-change triggers, or
`order.status-changed` for order-status triggers). Then look one event later
in the timeline for a `mutate` whose `json.automationResults[]` contains your
op.

```
GET /api/v2/platformModel/events
  ?recordId=REC_123
  &boardId=BRD_456
  &objectKey=load__c
  &limit=100
  &orderBy=createdAt
  &orderByDirection=desc
  &notInTypes=widget-open,widget-close,widget-skip
```

**Interpretation.** If the trigger event is missing, the upstream change you
*thought* happened never did — e.g., the user clicked a button that hit a
guarded API path, the field write was rejected by validation, or the actor
saved a different field than you expected. If the trigger event is present but
no follow-up `mutate` carries your op in `automationResults`, the automation
itself was never compiled into the recipe push (commonly: rejected at recipe
validation — see Common root causes below).

### Symptom 2 — "The automation ran but the action was skipped"

**Query:** same as above, then expand the `mutate` event and look at
`automationResults[].conditionResults`.

**Interpretation.** A falsy or empty `conditionResults` means the gate failed
*as designed*. Read the recipe's `condition` block for that op and reconcile it
against the record state at trigger time (`changedFields[].prevValue` shows
what the values were *before* the mutation). The most common surprise here is a
**cross-record condition read** — e.g.
`{"$all": {"load__c.status__c": {"$equals": "planning"}}}`. The platform does
*not* support cross-record reads inside conditions; the evaluator silently
no-fires the whole automation. Restructure the condition to test only fields on
the trigger record, or rely on `setValue` idempotency (re-setting a field to
its current value is a safe no-op) to push the cascade outward.

### Symptom 3 — "The automation ran but the write didn't take"

**Query:** same as above, then cross-reference each
`automationResults[].actionDetails.toPath` against the entries in
`json.changedFields[]`.

**Interpretation.** If the action ran (it appears in `automationResults` with a
populated `runResult`) but no matching `changedFields` entry exists, one of
three things happened:

1. **`toPath` pointed at a derived/LOOKUP field.** Derived fields are
   unwriteable; the engine reports a misleading "Unknown field 'X' in object
   'Y'" and aborts the *entire* automation. One bad action kills the whole op.
   Fix: never `copyValue`/`setValue` to a derived field. Populate the parent
   ref via `connectToPath` and let the LOOKUP resolve.
2. **The new value equals the previous value.** No-op writes don't show up in
   `changedFields`. Confirm by comparing `runResult.copiedValue` to the current
   field value via `GET /objects/{id}?paths=...`.
3. **A reverse array is missing.** `createRecord` with
   `connectToPath: "legs__c"` requires an `isArray: true` field on the parent;
   without it (or without `reverse: "<otherFieldKey>"` mirroring the back-ref)
   the child is created but the parent ref stays null.

### Symptom 4 — "The action seemed to fire but downstream state is wrong"

**Query:** events on **both** records — the trigger record and the record you
expected to be updated.

**Interpretation.** This is almost always a **cross-record write via
dot-traversal** issue. `copyValue` and `setValue` *do* support a dot-traversed
`toPath` (e.g. `"tractor__c.currentLocation__c"`), but with two gotchas:
(a) cross-record *reads* in `condition` silently no-fire the whole automation,
and (b) the search endpoint lags by minutes after a successful cross-record
write. If `automationResults[].runResult.copiedValue` is populated *and* the
target record's events show a `changedFields` entry, the write committed —
trust the events log over a stale search response. Use
`GET /objects/{id}?paths=field1,field2` for authoritative reads.

## Endpoint reference

```
GET {origin}/api/v2/platformModel/events
```

| Query param | Notes |
|---|---|
| `recordId` | Required for record-scoped debugging. |
| `boardId` | Required; pull from the record detail URL. |
| `objectKey` | Required; e.g. `load__c`, `leg__c`. |
| `limit` | Default 100. Paginate with `offset` for old/active records. |
| `orderBy` | Use `createdAt`. |
| `orderByDirection` | `desc` for newest first. |
| `notInTypes` | Always include `widget-open,widget-close,widget-skip` — they're noise. |

**Note:** the events endpoint **hard-excludes** `widget-open`, `widget-close`,
and `widget-skip` events at the server. You cannot include them with
`inTypes` — the filter is unconditional.

A typical paste-and-run query:

```
GET /api/v2/platformModel/events?recordId=REC_123&boardId=BRD_456&objectKey=load__c&limit=100&orderBy=createdAt&orderByDirection=desc&notInTypes=widget-open,widget-close,widget-skip
```

Headers: `Authorization: Bearer {token}` and `Content-Type: application/json`.
Use the bearer token from the Bloom CLI config or the `rr__token` cookie from
an authenticated tab.

## RecordEvent shape

```ts
RecordEvent {
  id, type, createdAt,
  user: { id, email, firstName },
  json: {
    actor?: { widgetKey, isUserSubmitted },
    logMessage?,
    changedFields?: [{ field, value, prevValue, type: { key, configKey } }],
    automationResults?: [{
      opName,
      actionDetails: {
        actionKey,
        toPath?, methodKey?, objectKey?, literalValue?, parameters?
      },
      conditionResults?,
      runResult?: { copiedValue? }
    }],
    $source?,
    recordId?, objectKey?, externalId?,
    orderId?, newStatus?,
    isTemplate?
  }
}
```

A few interpretation rules:

- **Automations show up as `type: "mutate"`.** Filter to those first.
- `actor.widgetKey === null` and `actor.isUserSubmitted === false` (or
  `json.$source === "automation"`) indicate a system-driven mutation rather
  than a user save.
- **Order-status automations trigger on `type: "order.status-changed"`**, not
  `mutate`. The resulting `mutate` events follow it in the timeline.
- `changedFields[]` is the ground truth for "what got written." If a field
  isn't there, it didn't change.
- `automationResults[].runResult.copiedValue` is the authoritative
  resolved-source value for `copyValue` actions.

## Eventual consistency

Events take **up to ~2 seconds** to land in the audit log. If you query
immediately after a save, you may see the trigger but miss the resulting
`mutate`. Retry with backoff (e.g., 500ms / 1.5s / 3s) before concluding the
automation didn't run. The search endpoint lags much longer (minutes) — never
use search to verify a cross-record write; use `GET /objects/{id}?paths=...`
instead.

## Common root causes

A surprising number of "didn't fire" cases are actually **recipe-rejected at
validation**, not silent runtime failures. The platform validates
`jsActionable.action.actionKey` against a closed discriminated union with
error `BRL-1003: no union branch matches for "<key>"`.

**Accepted (recipe-pushable) action keys:**
`setValue`, `copyValue`, `createRecord`, `removeValue`, `compositeAction`,
`connectToPath` (inside `createRecord`), `invokeMethod`, `sendNotification`.

**Rejected (closed-union, not recipe-pushable):**
`notify`, `showToast`, `sendEmail`, `mathExpression`, `callAction`, `loop`,
`removeRecord`. These either don't exist or require Builder UI to inject
runtime pieces the CLI can't ship.

When debugging, also remember:

- **`copyValue` of object-ref fields silently no-ops.** Scalar copyValue works;
  ref copyValue does not. Use `connectToPath` for ref wiring.
- **`createRecord` requires an inverse array field on the parent**
  (`isArray: true`) named in `connectToPath`. Without it, the child is
  created but the parent's array isn't appended.
- **`compositeAction` is all-or-nothing.** One bad sub-action (e.g., a write
  to a derived field) aborts the whole composite — including the writes that
  would otherwise have succeeded.

## Cross-references

- `automations.md` — full action-key catalog with shape examples.
- `api-platformmodel.md` — auth, base URLs, and shared response conventions.
