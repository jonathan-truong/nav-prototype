# Board Views API

Saved board views are named tabs layered on top of a base board. Each view carries its own filter combo, column ordering, sort, and (optionally) a charts layout.

**Saved views ARE recipe-managed via the configRow channel.** `recipe:pull` snapshots every view to `bloom-recipes/<orgId>/configRows/boardView/<boardId>/<slug>.json` (namespaced by parent board UUID), and `recipe:push` re-uploads them. The on-disk shape is:

```json
{
  "id": "<view-uuid>",
  "type": "boardView",
  "name": "Hot Leads",
  "json": {
    "id": "hotLeadsView",
    "boardId": "<board-uuid>",
    "layout": "table",
    "filters": [...],
    "filtersOperator": "and",
    "labelId": "Hot Leads",
    "columnIds": [...],
    "layoutOptions": null
  }
}
```

`json` is exactly the body you'd POST to the API endpoints below. So you have **two equivalent provisioning paths**: (a) author the file locally and `recipe:push`, or (b) POST directly via the API in this doc. This doc focuses on path (b) because it's how generative skills work and how one-off seed scripts run, but everything below maps to the configRow file under `json:`.

> **Why namespaced by boardId**: in production orgs, view names like "Reporting" appear on 20+ different boards. A flat layout (`configRows/boardView/reporting.json`) silently collapses them — every pull overwrites the previous board's "Reporting" view, dropping data. The `<boardId>/` namespacing eliminates the cross-board collision class. The boardId is a UUID rather than a human-readable board key because the boardId-to-board-key mapping isn't derivable client-side from the snapshot.

> **Same-board name collisions** still happen (rare): two views on one board with identical names get sorted-positional filenames — `reporting.json`, `reporting-2.json`, `reporting-3.json` — sorted by the server-issued view UUID for deterministic assignment. Identity lives in `{id, type}` inside each file, so positional naming is safe.

### Pre-push validation guards

Before `recipe:push` sends the payload, bloom locally guards three things on every configRow:

- **id must be a valid UUID.** Empty / missing / non-UUID ids are blocked with all offenders listed; no network call is made. Fix all in one pass instead of round-tripping.
- **No duplicate ids within the payload.** Each duplicate is reported.
- **`visibility="user"` triggers a warning** (not a block). The SDLC scope is org-wide so the row is still sent, and the server rejects it with `SDLC-1007: configRow.json.visibility must be one of board`. The pre-push warning makes the cause visible before the request fails.

### SDLC-1007 reject details

Server-side validation runs up front and surfaces every offender in a single `SDLC-1007 INVALID_CONFIG_ROW` 400 response with a structured `details.rejected: [{id, reason}]` array. The CLI renders that inline:

```
Push failed: SDLC-1007: ...
Rejected 2 configRow(s):
  - <uuid-1>: configRow.json.visibility must be one of board
  - <uuid-2>: configRow.type "fooView" is not supported. Allowed: boardView, childBoard
```

The full set of canonical reason strings (verbatim from the server):

- `configRow.id is required`
- `configRow.id must be a UUID`
- `configRow.id appears more than once in this payload`
- `configRow.type "<type>" is not supported. Allowed: boardView, childBoard`
- `configRow.json must be an object`
- `configRow.json.boardId is required for boardView rows`
- `configRow.json.visibility must be one of board`
- `configRow.type cannot change. Existing row has type "<prior>", payload sent "<new>".`

The only configRow types accepted are **`boardView`** and **`childBoard`**. Custom types are rejected.

> **Historical note (pre-fix behavior):** before the platform-side configRow fix shipped, brand-new `boardView` rows pushed via `/boreal/push` returned `written: 1` but were silently filtered from `/boreal/pull` because the server's snapshot read excluded rows where `refId2` (visibility) was NULL — and inserts left it NULL. The CLI's reconcile pass would then quarantine the file the user had just authored. If you have boardViews authored before the fix landed and they aren't returning via `recipe:pull`, re-push them and they'll round-trip cleanly. UI-created views were unaffected.

For the auth flow and shared header conventions, cross-link to `api-platformmodel.md`.

> **Canonical platform reference:** [https://roserocket.readme.io/docs/getting-started](https://roserocket.readme.io/docs/getting-started). All platform endpoints are versioned as `/api/v2/...`.

## Finding your boardId

Every endpoint here takes `{boardId}` as a UUID — **NOT** the recipe key. The UUID is not in your local `bloom-recipes/<orgId>/boards/<key>.json` file. Look it up at runtime:

```bash
curl -sS "https://<subdomain>.roserocket.com/api/v2/platformModel/boards/admin/all" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-org-id: $ORG_ID" | jq '.[] | select(.key=="leadBoard__c")'
```

Returns:

```json
{
  "id": "3988dee6-614c-5c13-8cf9-5b7bc6110dce",
  "key": "leadBoard__c",
  "labelId": "Leads",
  "objectKey": "lead__c",
  "objectLabel": "Lead",
  "isExternal": false
}
```

The `id` is your `boardId`. Cache it for the session.

`/boards/admin/all` is the **comprehensive** board endpoint — returns recipe boards, child/custom boards, hidden boards, and system internals. Prefer it over `/boards/nav` (which only returns sidebar-visible boards) for any scripting or tooling lookup. Requires admin role; staff-only on shared orgs.

> **Pre-publish caveat:** `/boards/admin/all` only returns boards already published to the live boards table. A brand-new board you've added to the recipe but not yet pushed/published won't appear yet. Push first, then re-query.

---

## TL;DR — A complete create payload

```http
POST /api/v2/platformModel/boards/{boardId}/views
Authorization: Bearer <bloom token>
x-org-id: <orgUuid>
Content-Type: application/json
```

```json
{
  "labelId": "Today's pickups",
  "description": "Loads with planned start in the next 24h, status not delivered",
  "boardId": "baceb893-2dfe-57e0-a8c9-6c79b99302ac",
  "id": "defaultView",
  "isDefault": false,
  "isPinned": true,
  "visibility": "board",
  "layout": "table",
  "layoutOptions": null,
  "columnIds": [
    "fullId",
    "load__c",
    "originCity__c",
    "destCity__c",
    "plannedStartAt__c",
    "driver__c",
    "status__c"
  ],
  "stickyColumnIds": ["fullId"],
  "mainLinkColumnKey": "fullId",
  "orderByDirection": "asc",
  "filtersOperator": "and",
  "filters": [
    {
      "filterKey": "status__c",
      "path": "status__c",
      "operator": "in",
      "value": ["planned", "dispatched", "in_transit"],
      "fieldType": "SelectField"
    },
    {
      "filterKey": "loadType__c",
      "path": "loadType__c",
      "operator": "in",
      "value": ["FTL"],
      "fieldType": "SelectField"
    },
    {
      "filterKey": "plannedStartAt__c",
      "path": "plannedStartAt__c",
      "operator": "between",
      "value": ["2026-05-03T00:00:00", "2026-05-04T00:00:00"],
      "fieldType": "DateField"
    }
  ],
  "createdAt": ""
}
```

The server responds with the same view shape, plus `id` (UUID), `ordinal` (tab position), `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, and `orgId`. **Capture the `id` locally** — every subsequent update or delete needs it.

---

## Endpoint and identifiers

| Verb | Path | Purpose |
|---|---|---|
| GET | `/api/v2/platformModel/boards/{boardId}/views` | List all views on a board (returns a bare array, not wrapped) |
| POST | `/api/v2/platformModel/boards/{boardId}/views` | Create a view, server assigns `id` and `ordinal` |
| PUT/PATCH | `/api/v2/platformModel/boards/{boardId}/views/{viewId}` | Update an existing view (use `viewId` from create response) |
| DELETE | `/api/v2/platformModel/boards/{boardId}/views/{viewId}` | Remove a view, returns 204 |

`boardId` **must be the board's UUID, not the board key**. To translate a key (e.g. `loadBoard__c`) into its UUID, hit `GET /api/v2/platformModel/boards/admin/all` (see "Finding your boardId" above) and look the key up in the returned list.

Auth uses the bloom CLI access token. Pull it from `~/Library/Preferences/bloom-cli-nodejs/config.json` → `auth.accessToken`, and pass the org UUID via the `x-org-id` header. See `api-platformmodel.md` for the full auth contract.

---

## Top-level view fields

| Property | Type | Notes |
|---|---|---|
| `labelId` | string | Display name on the tab |
| `description` | string | Tooltip / hover text |
| `boardId` | uuid | Must match the path param |
| `id` | string | Send `"defaultView"` as a placeholder on create — server returns the real UUID |
| `isDefault` | bool | If true, the view auto-opens when the board loads |
| `isPinned` | bool | Pinned views show in the tab bar without expansion |
| `visibility` | string | `"board"` (org-wide) is the only value accepted by SDLC `/push`. `"user"` is rejected with SDLC-1007 (`configRow.json.visibility must be one of board`) — user-scoped views are personal preferences and out of scope for SDLC sync. `"role"` is similarly not in the validator's allowed set |
| `layout` | string | `"table"` and `"charts"` confirmed. Kanban/list/calendar likely but untested |
| `layoutOptions` | object \| null | `null` for table; required for `"charts"` (see below) |
| `columnIds` | string[] | Ordered column keys; columns must exist on the base board |
| `stickyColumnIds` | string[] | Columns frozen left during horizontal scroll. Required even for charts views — leave as `["fullId"]` |
| `mainLinkColumnKey` | string | Column whose cells link into the underlying record. Required even for charts views |
| `orderByDirection` | string | `"asc"` or `"desc"` |
| `filtersOperator` | string | `"and"` or `"or"` — joins the entries in `filters[]` |
| `filters` | object[] | Filter expressions (see below). Empty array = no filters |
| `createdAt` | string | Send empty string on create; server populates |

---

## Filters

Each entry in `filters[]` has the shape:

```json
{
  "filterKey": "<field-key>",
  "path": "<field-path>",
  "operator": "<one-of-allowed-list>",
  "value": "<scalar | array | null>",
  "fieldType": "SelectField | TextField | DateField | ToggleField | ConnectionField | NumberField | DerivedField | ..."
}
```

### Allowed operators

The complete set, verified by API error responses:

| Operator | Value shape | Notes |
|---|---|---|
| `in` | array | Use this for SelectField equality — even single-value matches |
| `notIn` | array | Inverse of `in` |
| `equals` | scalar | Safe for TextField, NumberField, ToggleField. **NOT for SelectField** (see gotcha) |
| `notEquals` | scalar | Same caveat as `equals` |
| `contains` | string | Substring match on text fields |
| `startsWith` | string | Prefix match |
| `endsWith` | string | Suffix match |
| `hasEvery` | array | All listed values present (multi-value fields) |
| `hasSome` | array | Any listed value present (multi-value fields) |
| `empty` | null | `value` is required, pass `null` |
| `notEmpty` | null | Same as `empty` |
| `freeText` | string | Full-text search across the field |
| `between` | [from, to] | Two-element array. Verified for DateField |

### CRITICAL gotcha — SelectField filters need `in: [...]`, never `equals: scalar`

**SelectField filters MUST use `in` with an array, even when matching a single value. Using `equals` with a scalar saves successfully (the POST returns 200) but FAILS at runtime in two ways: (1) live-tested on `POST /objects/search` returns `500 Internal Server Error` outright; (2) when the broken view loads in the UI, results may render empty with no visible error.** Either way the saved view is unusable. This is the single most common way a saved view becomes broken without any visible authoring-time signal.

```json
// Correct — works AND filters at runtime
{
  "filterKey": "status__c",
  "path": "status__c",
  "operator": "in",
  "value": ["paid"],
  "fieldType": "SelectField"
}

// Wrong — POST returns 200, view persists, but at runtime returns "No matching results found"
{
  "filterKey": "status__c",
  "path": "status__c",
  "operator": "equals",
  "value": "paid",
  "fieldType": "SelectField"
}
```

The wrong form passes server validation, persists to the DB, and renders in the UI with the operator dropdown showing `--` instead of a real operator. The platform's runtime query layer can't translate `equals + scalar` against a SelectField column. Verified live on ryancrm: hitting `POST /api/v2/platformModel/objects/search` with a SelectField+equals filter returns `{"statusCode":500,"message":"Internal server error"}`. The view loads as a broken tab; users see no records (and no error) until they click the filter chip.

**Empirical rule:** for any "is one of" semantics on enum-style fields (Select, MultiSelect, Status), always use `in: [...]`. Don't use `equals` for select fields — period. If you have a provisioning script, audit it for this pattern; one mistake can break dozens of views silently.

### DerivedField (LOOKUP) filters

Lookup-style derived fields use `fieldType: "DerivedField"` even when the underlying value is a select enum, text, or number. The fieldType drives how the platform indexes the field server-side, so passing the underlying type instead (e.g., `"SelectField"` for a derived select column) leads to the same silent-fail symptom: the view persists with `labelId: null` and `ordinal: null`.

> **Search vs saved-view filter parity gap.** Filter shapes that work as **persisted saved-view filters** (i.e. stored on the view via this API and applied at runtime when the user opens the tab) are NOT all accepted as direct **`POST /objects/search`** payloads. Verified live on ryancrm: `{fieldType:"DateField", operator:"between", value:[...]}` and `{fieldType:"DerivedField", operator:"in", value:[...]}` both return `500 Internal Server Error` from the search endpoint, even with the exact filter shape carrier106 stores in production saved views (e.g. `plate expiring (90d)`). Confirmed working on `/objects/search`: `SelectField + in`, `ConnectionField + in/empty/notEmpty`, `TextField + freeText/contains`, `ToggleField + equals`. Confirmed broken: `DateField + between`, `DerivedField + in`. Workaround for ad-hoc queries that need date or derived-field filtering: persist the filter on a saved view (where it works at UI runtime) and read records via the saved view rather than direct search; or fetch a wider set and filter client-side.

Operator and value semantics still mirror the underlying type:

```json
{
  "filterKey": "customerCreditStatus__c",
  "path": "customerCreditStatus__c",
  "operator": "in",
  "value": ["approved", "approved_with_limit"],
  "fieldType": "DerivedField"
}
```

`contains` works for derived text columns, `between` for derived date columns, etc.

### Date range (`between`)

```json
{
  "filterKey": "deliveredAt__c",
  "path": "deliveredAt__c",
  "operator": "between",
  "value": ["2026-04-01T00:00:00", "2026-04-30T23:59:59"],
  "fieldType": "DateField"
}
```

Use ISO-8601 strings. Both bounds are required.

### Other field-type cheat sheet

- TextField + `contains`/`startsWith`/`endsWith`/`equals`: scalar string value
- ToggleField: `{operator: "equals", value: true | false, fieldType: "ToggleField"}`
- ConnectionField + `empty`/`notEmpty`: `value: null`
- NumberField + `equals`/`between`: scalar or [from, to]

---

## Charts layout (canonical shape)

A chart view stores TWO independently-shaped objects under `layoutOptions`:

```json
"layoutOptions": {
  "chartsConfig": [ /* array of chart specs */ ],
  "panelsConfig": { /* grid layout that points at chart specs by id */ }
}
```

**`chartsConfig`** is an array of `ChartConfig` objects:

```json
{
  "id": "chart-1",
  "type": "bar",  // "bar" | "metric" | "donut" (verified). "line", "area", etc. exist in the type union but are out-of-scope for this doc.
  "label": "",
  "query": {
    "formula": { "operator": "count" },  // or {"operator":"countUnique"|"sum"|"avg"|"min"|"max", "columnKey":"<field>"}. count omits columnKey.
    "groupBy": { "columnKey": "status__c", "dateGranularity": null },  // dateGranularity: "day"|"week"|"month"|"quarter"|"year" for date columns
    "stackBy": { "columnKey": "customer__c" }  // optional; only meaningful for bar
  },
  "displayOptions": { "orientation": "vertical" }  // bar uses orientation; metric uses {caption:"..."}; donut uses {}
}
```

**`panelsConfig`** is the grid layout. Panels reference charts by `chartId`; chart specs do NOT live inside panels:

```json
{
  "direction": "vertical",
  "groupSizes": { "group-1": 50, "group-2": 50 },
  "panelGroups": [
    {
      "id": "group-1",
      "direction": "horizontal",
      "panelSizes": { "panel-1": 50, "panel-2": 50 },
      "panels": [
        { "id": "panel-1", "chartId": "chart-1" },
        { "id": "panel-2", "chartId": "chart-2" }
      ]
    }
  ]
}
```

`groupSizes` and `panelSizes` are optional percentages (auto-distribute if omitted).

> **Common mistake**: flattening chart definitions into panels (`{type, field, aggregation, ...}` directly on the panel object). The runtime expects `charts[]` to be an array — a flat panel-shape produces `TypeError: charts.map is not a function` when the view loads, with a generic "this page is having issues" toast in the UI.

**Canonical reference**: production chart views in the `carrier106` org (e.g. `loadBoard__c`, `legBoard__c`, `tractorBoard__c`). Pull one via `GET /boards/<boardId>/views/<viewId>` and inspect `layoutOptions` to see a working shape for any chart type. The platform source-of-truth for the type definitions is `platform-components/ui/src/scripts/platform/components/BoardCharts/boardCharts.constants.ts`.

## Charts layout (high-level)

Set `layout: "charts"` and put chart definitions in `layoutOptions.chartsConfig`, with grid arrangement in `layoutOptions.panelsConfig`. Verified chart types: `bar`, `metric`, `donut`. Pie/line/area are likely but unverified.

```json
"layout": "charts",
"layoutOptions": {
  "chartsConfig": [/* array of chart objects */],
  "panelsConfig": {/* grid arrangement */}
}
```

Charts respect the view's top-level `filters[]` — useful for "this week's load count" by combining a `between` date filter with a `count` metric chart.

For chart specifics (formula operators, groupBy granularities, panel layout rules), use the `/bloom-generate-chart-view` skill or refer to a dedicated charts doc once one exists. Keep this reference focused on the table-view path.

---

## Relative-date filters

Relative-date tokens (e.g. "today + 90 days", "this week", "last 30 days") almost certainly exist somewhere in the platform — the board UI exposes them — but they are **untested in writeable form via this API**. If you POST a string like `"today"` or `"+90d"` as a `between` bound, the request may either error or persist a literal string that doesn't resolve.

**Workaround:** compute absolute ISO-8601 dates at provision time and re-run the seed script (or a scheduled job) on whatever cadence keeps the windows fresh. For most workflow tabs (e.g. "next 7 days") a daily re-provision is acceptable; for views that should always reflect "today" without a job, either accept staleness or wait until relative-token support is verified.

---

## No GET-by-id for views — use list GET

`GET /api/v2/platformModel/boards/{boardId}/views/{viewId}` returns 404 — there is no single-view-by-id read endpoint. Use `GET /api/v2/platformModel/boards/{boardId}/views` (list) and filter by `id` to retrieve a specific view's current state. Verified live on ryancrm.

(PUT and DELETE on `/views/{viewId}` work fine — only the GET-by-id is missing.)

## Idempotence and re-applies

There is no upsert. POST always creates a new view; the server assigns a fresh UUID and increments `ordinal`. To make your provisioning script idempotent:

1. After creating a view, persist its `(boardId, labelId) -> viewId` mapping to a local file or a recipe-adjacent state store.
2. On re-run, look up the mapping. If a viewId exists, PUT to `/views/{viewId}` with the new payload. If not, POST.
3. To reset a board's views, GET the array first, DELETE each `viewId`, then POST the new set.

If you skip the mapping and re-POST, you will accumulate duplicate tabs. The board UI does not deduplicate by name.

---

## Silent-fail symptom checklist

If a POST returns 200 but the view comes back with `labelId: null` and `ordinal: null`, the server rejected the filters but didn't surface a top-level error. Things to check, in order:

1. Did you use `in: [...]` (not `equals`) for every SelectField filter?
2. Did you use `fieldType: "DerivedField"` for every LOOKUP column?
3. Is `boardId` the UUID, not the board key?
4. Are all `columnIds` actually present on the base board?
5. Is the operator name spelled exactly as in the table above (`empty`, not `isEmpty`)?

The platform may also return a 400 with a `message` array enumerating allowed values when something is grossly wrong — useful for probing unknown operators.

---

## Cross-references

- Auth, headers, and `x-org-id` resolution: `api-platformmodel.md`
- Charts layout details: `/bloom-generate-chart-view` skill
- Why this isn't recipe-managed: views are an org-level config stored separately from board definitions. Base boards (columns, default filters, buttons, layout defaults) ship via recipes; named saved views layer on top via this API.
