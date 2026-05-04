# Platform Model API

REST reference for direct CRUD on platformModel records via the Rose Rocket public API. This is the JSON-API path used by tooling (including bloom-emitted skills) to seed sample data, query records, and debug. TS-recipe paths in `src/recipes/` are an escape hatch for bulk transforms; for ordinary record operations, prefer this API.

For saved-view creation, see `api-board-views.md`. For automation-run inspection (the `/events` API), see `debugging-automations.md`.

> **Canonical platform reference:** [https://roserocket.readme.io/docs/getting-started](https://roserocket.readme.io/docs/getting-started). Cross-check endpoint shapes there when something in this guide doesn't match what you observe — the Rose Rocket platform team maintains it. All platform endpoints are versioned as `/api/v2/...`.

---

## Endpoint summary

| Verb   | Path                                              | Body                              | Notes                                                                                  |
| ------ | ------------------------------------------------- | --------------------------------- | -------------------------------------------------------------------------------------- |
| POST   | `/api/v2/platformModel/objects`                   | `{objectKey, json: {...}}`        | Create. Returns `{id, fullId, errors, ...}`. Partial `errors` do NOT block creation.  |
| GET    | `/api/v2/platformModel/objects/{id}?paths=a,b,c`  | —                                 | Read by id. Without `paths`, connection fields are stripped from the response.         |
| PUT    | `/api/v2/platformModel/objects/{id}`              | `{objectKey, json: {...}}`        | Update. Replaces listed fields. PATCH is NOT supported (404).                          |
| DELETE | `/api/v2/platformModel/objects/{id}`              | —                                 | Returns 200 with the deleted record.                                                   |
| POST   | `/api/v2/platformModel/objects/search`            | `{boardId, filters, pagination}`  | Search. `boardId` MUST be a UUID — board key fails 400.                                |

All hosts are **production** (e.g. `https://<orgSubdomain>.roserocket.com`). Preview hosts return `Jwt issuer is not configured` because the JWT is issued by production `a.roserocket.com`.

### Headers (every request)

```
Authorization: Bearer <accessToken>
x-org-id: <orgUuid>
Content-Type: application/json
```

### Hosts

Both of these resolve correctly for any active org you're authenticated to:

- `https://network.roserocket.com` — multi-tenant entrypoint; uses `x-org-id` header to scope the request
- `https://<orgSubdomain>.roserocket.com` — subdomain-scoped (e.g. `https://ryancrm.roserocket.com`)

Do NOT hit `https://a.roserocket.com` for API calls — that's the JWT issuer and only serves `/authorize` and `/oauth/token`. Hitting it for object endpoints returns 404.

---

## Auth token

Bloom CLI persists its OAuth bearer at:

```
~/Library/Preferences/bloom-cli-nodejs/config.json
```

Read `auth.accessToken`:

```bash
jq -r .auth.accessToken ~/Library/Preferences/bloom-cli-nodejs/config.json
```

- **TTL:** 30 minutes. If a request returns 401, run any `bloom` command (e.g. `bloom recipe:pull --yes`) to refresh the token in place.
- **Issuer:** the JWT is issued by **production** `a.roserocket.com`, not `a.preview.roserocket.io`. Always hit production hosts.
- **Org binding:** the stored `orgId` must match the target org. Run `bloom org:switch <subdomain>` first if you're crossing orgs.

---

## POST envelope (required)

Every create/update body uses a two-key envelope. Top-level field properties (outside `json`) are rejected with `400 {"message": ["json must be an object"]}`.

```json
{
  "objectKey": "rateConfirmation__c",
  "json": {
    "name": "RC for Acme",
    "status__c": "draft",
    "lineHaulRate__c": { "amount": 125000, "currencyCode": "USD" }
  }
}
```

The `objectKey` is the recipe-side object identifier (e.g. `load__c`, `leg__c`, `carrier__c`). All field writes go inside `json`.

---

## Nested record creation (single POST creates parent + children)

Connection-field values can be **either** an existing-record ref `{id: "<uuid>"}` **or** a brand-new record's full JSON body. When the value is a body (no `id`), the platform creates the child record inline and wires the connection.

```json
POST /api/v2/platformModel/objects
{
  "objectKey": "opportunity__c",
  "json": {
    "name": "Acme Renewal",
    "stage__c": "qualifying",
    "account__c": { "id": "0c7f6ef4-509c-4dba-942f-352468d194bb" },
    "primaryPerson__c": {
      "name": "Inline Joe",
      "email__c": "joe@inline.example",
      "title__c": "VP"
    },
    "amount__c": { "amount": 1000000, "currencyCode": "USD" }
  }
}
```

Verified live: the POST above returns the new opportunity with `primaryPerson__c` ref auto-wired to a freshly-created person record (e.g. `PRS-14`). The person row is searchable immediately. Lookup-derived sibling fields (e.g. `primaryPersonName__c`) populate from the inline data on the same response.

Use when: bootstrapping a record graph in a single API call (lead → first-touch person, opportunity → primary contact, etc.) without round-tripping create-and-then-link.

## Field write shapes

The API is strict about per-type shape. Sending the wrong shape returns 400 with a parser error or 422 with the field's expected enum echoed back; `measurement` violations return BRL-1010.

| Type                    | Write shape                                                                                                | Notes                                                                                       |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `text`                  | scalar string                                                                                              | —                                                                                           |
| `url`                   | scalar string                                                                                              | —                                                                                           |
| `number`                | scalar number                                                                                              | —                                                                                           |
| `toggle`                | scalar boolean                                                                                             | —                                                                                           |
| `date`                  | ISO date string `"YYYY-MM-DD"`                                                                             | Bare date, no time component.                                                               |
| `dateTime`              | `{dateTimeInLocation: "YYYY-MM-DDTHH:MM:SS", dateTimeInLocationEnd: null}`                                 | **No `Z` suffix and no offset.** Bare ISO with `Z` returns 400.                             |
| `money`                 | `{amount: <integer cents>, currencyCode: "USD"}`                                                           | Field name is `currencyCode`, NOT `currency`. `amount` is integer minor units.              |
| `multiCurrencyMoney`    | **BROKEN — do not use**                                                                                    | Returns 500 on every write shape we tried; renders read-only in detail views. Use `money`.  |
| `measurement`           | scalar number                                                                                              | Field's `typeConfig` MUST set `unitCategory`, `defaultUnitType`, `currentUnitType`.        |
| object ref               | `{id: "<record-uuid>"}`                                                                                    | Bare UUID string fails: `Expected object for value`. Readback expands to the full ref.      |
| `select` / `multiSelect` | exact value(s) from `typeConfig.options`                                                                   | Server echoes valid values on 400/422.                                                      |

### dateTime — common gotcha

```json
"pickupEarliestAt__c": {
  "dateTimeInLocation": "2026-04-26T08:00:00",
  "dateTimeInLocationEnd": null
}
```

A bare `"2026-04-26T08:00:00Z"` returns:

```
should be a valid date in ISO 8601 format: YYYY-MM-DDTHH:MM:SS
```

Use `dateTimeInLocationEnd` for range fields; pass `null` for point-in-time.

### money — `currencyCode`, not `currency`

```json
"lineHaulRate__c": { "amount": 125000, "currencyCode": "USD" }
```

`amount` is an integer in minor units (cents). The key is literally `currencyCode` (ISO-4217). `currency` is silently ignored or 500s.

### multiCurrencyMoney — broken, replace it

The `multiCurrencyMoney` field type:

1. Renders **read-only** in the UI detail view — no edit affordance.
2. Returns 500 on the API for every shape tried (`{amount, currency}`, `{value, currency}`, `{amount: {USD: N}}`, `{USD: N}`, bare integer).

Migrate any `multiCurrencyMoney` field to `type: "money"` (with `typeConfig: {canBeNegative: false}`); pair with a sibling `select` field if you need an explicit per-record currency choice. Note: changing the type nulls existing values — plan a re-seed step.

### measurement — three required typeConfig keys

A `measurement` field MUST declare these three keys in its recipe-side `typeConfig`, or pushes fail with `BRL-1010 Missing required field ... in typeConfig`:

```json
{
  "type": { "key": "measurement", "ref": true, "configKey": "type" },
  "typeConfig": {
    "unitCategory": "length",
    "defaultUnitType": "feet",
    "currentUnitType": "feet",
    "precision": 2
  }
}
```

Use the **full word**, not the symbol: `feet` not `ft`, `pound` not `lb`. For `length`: `centimeter, inch, meter, feet, kilometer, mile`. For other categories, follow the same convention and let the validation error echo `expected`.

### object refs — must be `{id: ...}`

```json
"billToCustomer__c": { "id": "5b9c1d4e-..." }
```

A bare string `"5b9c1d4e-..."` returns `Expected object for value`. On readback the server expands to `{id, objectKey, orgId, name, fullId}`, but on write only `id` is required.

### select / multiSelect — exact match required

The value must match one of the options in the field's `typeConfig.options` exactly. On a mismatch the server returns 400/422 with the expected enum list in the error body — use that list verbatim.

---

## GET semantics

```
GET /api/v2/platformModel/objects/{id}?paths=fullId,name,originFacility__c,billToCustomer__c
```

- **Default:** connection fields (object refs, lookups, reverse-connections) are **stripped from the response**. They are not `null` — they're simply absent.
- **`paths` query param:** comma-separated field keys (NOT bracket-array). Pass every connection field you need expanded.
- Working: `?paths=field1,field2`. Not working: `?paths[]=...`, `?include=...`.

This is the authoritative read for verifying connection-field state after a write.

---

## Search

```json
POST /api/v2/platformModel/objects/search
{
  "boardId": "3988dee6-614c-5c13-8cf9-5b7bc6110dce",
  "filters": [
    {
      "filterKey": "fullId",
      "path": "fullId",
      "operator": "equals",
      "value": "LD-9",
      "fieldType": "TextField"
    }
  ]
}
```

- `boardId` MUST be a **UUID**. The board's recipe-side key (e.g. `leadBoard__c`) returns 400 with `["boardId should not be empty","boardId must be a UUID"]`.
- LOOKUP-derived fields auto-resolve in search results (when the index is current).

### Discovering boardIds

The board's UUID is **not** the same as its recipe key, and isn't stored in the local recipe JSON. Look it up at runtime:

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
  "icon": "🎯",
  "objectKey": "lead__c",
  "objectLabel": "Lead",
  "isExternal": false,
  "isHidden": null,
  "createdAt": "..."
}
```

Cache the `id` keyed by board key for the session. The UUID is stable across pulls but is NOT in `bloom-recipes/<orgId>/boards/<key>.json`.

**Why `/boards/admin/all` and not `/boards/nav`?** `/admin/all` returns the **complete board inventory** — recipe boards, child/custom boards, hidden boards, and system internals. `/boards/nav` returns only the subset visible in the left-rail navigation (a smaller, role-filtered list). For tooling that needs to find a board by key (the typical scripting case), `/admin/all` is the canonical lookup. Use `/nav` only when you want what the user actually sees in the sidebar. Requires admin role; staff-only on shared orgs.

> **Pre-publish caveat:** even `/boards/admin/all` only returns boards already published to the live boards table. A brand-new board you've added to the recipe but not yet pushed/published won't appear here. Push first, then re-query.

---

## Derived fields cannot be written

Any field whose value is computed by a formula or LOOKUP rejects writes with 400:

```
Error setting field <key>: derived field cannot be written
```

Example: `loadInvoice.totalAmount__c` is computed from `load.lineHaulRate__c + fuelSurcharge + accessorialTotal`. Don't POST/PUT `totalAmount__c` directly — set the source components on the parent load and the LOOKUP cascades. Same rule for any `text`-cache-of-LOOKUP, formula field, or aggregate.

---

## PUT and automations — verified to fire

PUT-via-API DOES fire automations on the same event semantics as UI updates. Verified live on ryancrm: PUT-updating an opportunity's `stage__c` from `prospecting` → `qualifying` fired `setProbabilityForQualifying` synchronously, the response payload showed `probability__c: 20` immediately, and the `/events` API recorded the `mutate` event with a populated `automationResults[].opName` of `opportunity__c.setProbabilityForQualifying.invokeAutomationAction.setValue`.

There IS one historical case where a PUT bypassed an automation chain (carrier104: PUT-setting `originFacility__c` / `destFacility__c` on a load did NOT fire `createPickupStopOnLoad` / `createDeliveryStopOnLoad`). That looked like an automation bug specific to those methods, not a general "PUT skips automations" rule. Trust automations to fire by default; if a chain doesn't trigger, query `/events` to confirm and treat the missing automation as a bug to investigate, not a known platform behavior.

For inspecting which automations actually fired on a record, use the `/events` API — see `debugging-automations.md`.

---

## Eventual consistency — search lags, GET is authoritative

`POST /objects/search` is backed by Elasticsearch and has indexing lag. Records and connection fields written **seconds (sometimes minutes) ago** may return zero rows or stale `null` even with `filters: []`.

Verification rules:

- **Did the write succeed?** Trust a 200/201 response with a non-empty `id`/`fullId` in the body. The POST/PUT response includes the full record with resolved refs and computed LOOKUPs.
- **Did a connection field actually get the new value?** `GET /objects/{id}?paths=fieldA,fieldB` — never search.
- **Did downstream automations run?** Hit the `/events` API (see `debugging-automations.md`); search results lie until indexing catches up.
- **Chaining writes in a script** (e.g. load → leg by load.id): pass the `id` from the create response forward; never re-search for it.

---

## DO NOT push to GTI-Canada

The Rose Rocket org **GTI-Canada — Internal Testing** (org UUID `632c4a4e-5843-4626-9535-6e5dfcd2d55f`) is a **read-only reference** for widget, widget-sequence, and object examples. Never POST, PUT, or DELETE against it. Specifically forbidden:

- `POST /api/v2/platformModel/objects`
- `PUT /api/v2/platformModel/objects/{id}`
- `DELETE /api/v2/platformModel/objects/{id}`
- `bloom recipe:push`, `bloom recipe:publish`
- `PUT /cookbook/.../configs`, `PATCH /cookbook/.../publish`, `DELETE /cookbook/.../configs/...`

If you `bloom org:switch` to GTI for inspection, switch back **immediately** before any write-shaped command. For read-only inspection prefer `GET /api/v2/platformModel/recipes/current/*?composed=true`, which doesn't require an org switch. When pulling for reference, use `bloom recipe:pull --output-dir <side-location>` so results land outside this project's `bloom-recipes/` tree.

---

## Quick reference — minimal create

```bash
TOKEN=$(jq -r .auth.accessToken ~/Library/Preferences/bloom-cli-nodejs/config.json)
ORG_ID=$(jq -r .auth.orgId      ~/Library/Preferences/bloom-cli-nodejs/config.json)

curl -sS -X POST "https://<subdomain>.roserocket.com/api/v2/platformModel/objects" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-org-id: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "objectKey": "rateConfirmation__c",
    "json": {
      "name": "RC for Acme",
      "lineHaulRate__c":   { "amount": 125000, "currencyCode": "USD" },
      "pickupEarliestAt__c": { "dateTimeInLocation": "2026-05-01T08:00:00", "dateTimeInLocationEnd": null },
      "billToCustomer__c": { "id": "5b9c1d4e-..." }
    }
  }'
```

Read it back with connection fields expanded:

```bash
curl -sS "https://<subdomain>.roserocket.com/api/v2/platformModel/objects/<id>?paths=fullId,name,lineHaulRate__c,pickupEarliestAt__c,billToCustomer__c" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-org-id: $ORG_ID"
```
