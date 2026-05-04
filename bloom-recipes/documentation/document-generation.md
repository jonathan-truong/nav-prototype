# Custom Document Generation

End-to-end guide to **custom document types and template-driven PDF generation** in Rose Rocket. This area of the platform is otherwise undocumented; what follows is the result of end-to-end probes against live carrier orgs.

The breakthrough — and the headline of this doc — is that **custom-object PDF generation does work via the platformModel API**, but only with a specific write shape that combines `isSystemGenerated` + `systemDocParentId`. Field-name guesses fail silently, the `isAutoCreated` flag does not actually fire on custom objects, and several adjacent paths (`getRecordDocuments`, `document.recordId`) drop data on the floor. The recipe in step-by-step form is at the bottom — read the architecture first so the platform gaps make sense.

For platformModel CRUD basics see `api-platformmodel.md`. For the workflow engine that fills the auto-create gap see `workflows-rocket-engine.md`.

---

## The three-layer architecture

Custom document generation is split across three distinct layers. Each layer has its own object/config and its own write path; getting any one wrong leaves the other two looking correct but rendering nothing.

| Layer | Object / Config        | Where it lives                                          | Purpose                                                                                                   |
| ----- | ---------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 1     | `documentType` config  | Org's `<orgId>ObjectBuilderMixin` recipe                | Registers a doc type `X` against an objectKey `Y`. Recipe-only — does NOT extend the closed platform enum. |
| 2     | `documentTemplate` rec | platformModel record                                    | Holds the Handlebars HTML body in the `templateData` field plus `rootObjectKey` binding.                  |
| 3     | `document` record       | platformModel record                                    | Per-instance generated artifact. Has `externalUrl` pointing at the `/pdf` render endpoint.                |

All three need to line up for end-to-end generation. The most common failure mode is creating layers 1 + 2 correctly and then not knowing the right shape for layer 3 — which is what this doc fixes.

---

## Layer 1 — registering a custom document type

```http
POST /api/v2/platformModel/documents/customType
Content-Type: application/json

{
  "label": "Bill of Lading",
  "objectKey": "trailer__c",
  "description": "Trailer BOL",
  "isCustomizable": true,
  "isAutoCreated": true,
  "isUserSubmitted": true
}
```

Returns 201 with the published recipe descriptor. The platform persists this to:

```
<orgId>ObjectBuilderMixin/configs/documentType/<key>
```

…where `<key>` is derived from `label.toLowerCase().replace(/\s+/g,'_') + '__c'` (for example `bill_of_lading__c`).

**Soft delete**: re-POST with `isDisabled: true`.
**Hard delete**: `DELETE /api/v2/platformModel/cookbook/<recipe>/configs/documentType/<key>`.

> **Platform gap — closed `documentType` enum.** The built-in enum (`billOfLading`, `shipmentBillOfLading`, `commercialInvoice`, `rateConfirmation`, `paps`, `labels`, `invoice`, `offloadManifest`, `preloadManifest`, etc.) is **not** extended by `/documents/customType`. Custom values are recipe-only — they round-trip through the documentTemplate / document write path, but they will not appear in some platform UI dropdowns.

---

## Layer 2 — creating a documentTemplate

```http
POST /api/v2/platformModel/objects
Content-Type: application/json

{
  "objectKey": "documentTemplate",
  "json": {
    "name": "Trailer BOL",
    "documentType": "bill_of_lading__c",
    "rootObjectKey": "trailer__c"
  }
}
```

The response includes `id`, `fullId` (e.g. `DOCT-7`), and `editorUrl: "_/#/ops/document-builder-admin/<id>"`.

### Setting the HTML body — `templateData`, NOT `body`

Probably the single biggest time sink in this area: **the field that holds the Handlebars HTML body is `templateData`**. Every other plausible name silently drops on the server side:

| Field name attempted                                                                           | Behavior                |
| ---------------------------------------------------------------------------------------------- | ----------------------- |
| `body`, `template`, `html`, `content`, `templateBody`, `dataFormat`, `blocks`, `pages` | Silently dropped on PUT |
| `templateData`                                                                                 | Persists                |

```http
PUT /api/v2/platformModel/objects/<template-id>
Content-Type: application/json

{
  "objectKey": "documentTemplate",
  "json": {
    "templateData": "<div>Hello World — {{record.fullId}}</div>",
    "isCodeMode": true,
    "showPageNumbers": false
  }
}
```

`isCodeMode: true` enables Handlebars / raw-HTML editing in the Document Builder UI Code panel.

### `rootObjectKey` coercion

> **Platform gap — silent rootObjectKey coercion on create.** POSTing a documentTemplate with a custom `rootObjectKey` may **silently coerce to `"order"`** on the initial create call (the closed `documentType` enum drives this — sending one of the base enum values, e.g. `shipmentBillOfLading`, will overwrite `rootObjectKey` to `shipment`). After the create, **GET the record back and verify `rootObjectKey`**; if it was coerced, do a follow-up PUT with the desired `rootObjectKey` — that path is not coerced.

Also note: `rootObjectLabel` is **derived** — sending it returns 400 `Error setting field`.

### `isDefault` is NOT settable via PUT — needs a separate action

```http
PUT /api/v2/platformModel/objects/<template-id>
{ "objectKey": "documentTemplate", "json": { "isDefault": true } }
→ 400 "Error setting field: isDefault"
```

Use the dedicated action endpoint instead:

```http
POST /api/v2/platformModel/actions/documentTemplate/setOrgDefault/1.0
Content-Type: application/json

{
  "recordId": "<template-id>",
  "boardId": "<documentTemplate-board-uuid>",
  "widgetKey": null,
  "isUserSubmitted": null
}
```

Returns 201 with the updated record (`isDefault: true`). After this, all future renders for this org against `documentType: <type>` use this template.

---

## Layer 3 — creating a document instance (the breakthrough shape)

This is the part of the API that is **not** documented anywhere else, that fails silently on every adjacent guess, and that took an end-to-end probe to nail down. Use this exact shape:

```http
POST /api/v2/platformModel/objects
Content-Type: application/json

{
  "objectKey": "document",
  "json": {
    "documentType": "bill_of_lading__c",
    "isSystemGenerated": true,
    "systemDocParentId": "<parent record uuid>"
  }
}
```

The two load-bearing fields are:

- `isSystemGenerated: true` — flips the doc into "rendered from template" mode, instead of "uploaded blob" mode.
- `systemDocParentId` — the parent record UUID against which Handlebars resolves `{{record.*}}`.

The response contains:

- `id` — the new document record id
- `externalUrl: "/api/v2/platformModel/documents/<doc-id>/pdf"` — the **direct render URL**

GET that `externalUrl` and the response is `application/pdf` rendered from the org's active-default template for the documentType, with the parent record as Handlebars context.

```http
GET /api/v2/platformModel/documents/<doc-id>/pdf
Accept: application/pdf
```

### Verified end-to-end

Verified 2026-04-29 against carrier106. Created a `bill_of_lading__c` doc with `systemDocParentId: <trailer-id>`. The rendered PDF resolved `{{record.fullId}}`, `{{record.unitNumber__c}}`, and `{{record.status__c}}` correctly against the trailer record — even though `trailer__c` is a custom `__c` object outside the base `documentType` enum. The breakthrough is real; the field names are fragile.

### What looks similar but does NOT work

- **`recordId` and `objectKey` (on the document body)** are silently dropped on create. POST with `{recordId: "<trailer-id>", objectKey: "trailer__c"}` returns 201, but GET shows `recordId: null` and `objectKey: "document"` (the doc's own self-key). The platform's internal `documentService.create()` is the only path that sets these; the public write API does not expose them. Consequence: the Documents embedded board (which filters by `recordId`) does **not** display API-created docs.
- **The `presignedUrl` endpoint** (`/api/v2/platformModel/documents/presignedUrl/<id>`) is an **upload** endpoint, not a render endpoint. Don't confuse it with `/<id>/pdf`.
- **A non-`isSystemGenerated` document** is a blank/pending record — its `externalUrl` returns `{message: "Document has no url or file"}` until something either uploads bytes via presignedUrl or flips it to system-generated.

---

## Auto-create automation: the platform gap, and the workaround

You might expect that registering a customType with `isAutoCreated: true` would auto-spawn a document on parent record creation, the way base `shipment` auto-populates `shipmentBillOfLadingUrl`. It does not.

> **Platform gap — `isAutoCreated` is a no-op for custom objects.** Verified 2026-04-29: a fresh trailer create with `isAutoCreated: true` registered + a default template set produced **zero** doc-related fields on the trailer and **zero** document records. Compare with base `shipment`, where the BOL doc is created synchronously.

JSON recipe automations also fail. The naive shape:

```json
{
  "actionKey": "createRecord",
  "objectKey": "document",
  "initialValueActions": [
    { "toPath": "documentType", "actionKey": "setValue", "literalValue": "bill_of_lading__c" }
  ]
}
```

…fails because **the Bloom automation engine validates `toPath` against the trigger object's schema, not the target object's schema**, when the target is `document`. The events API surfaces:

```
"error": "Unknown field 'documentType' in object 'trailer__c'"
"opName": "trailer__c.createDocOnTrailerCreate.invokeAutomationAction.createRecord"
```

`invokeMethod` with arbitrary `methodKey` (`createForRecord`, `generateForRecord`, etc.) silently no-ops — the platform accepts any string and does nothing.

### Workaround — Rocket Engine workflow

The supported path is a **Rocket Engine workflow** (cross-link: `workflows-rocket-engine.md`). Workflow steps run server-side against the platformModel API, so they can issue the verified `{isSystemGenerated, systemDocParentId}` POST shape directly.

1. Trigger: parent record `onCreate`.
2. Guard: `$isEmpty` check on (e.g.) the parent's `documents__c` connection — prevents re-firing on idempotent retries.
3. Step: `workflowUtils/createRecord/1.0` with the verified `document` shape, passing `systemDocParentId: $context.record.id`.

The workflow runs as the platform user, so the `document` write path treats it the same as the manual API call — `isSystemGenerated` and `systemDocParentId` both persist, and the resulting `externalUrl` renders correctly.

---

## The `$custom_documentTemplate` mixin cache gotcha

While iterating you may hit a state where new customType registrations don't take effect on the documentTemplate enum, even after deletion + republish.

> **Mixin cache gotcha — `$custom_documentTemplate` freezes the documentType enum.** While the mixin file is present, the `documentTemplate.documentType` Zod enum is **closed** to base values. To unblock changes:
>
> 1. Delete the mixin file from your local recipe directory (otherwise `bloom recipe:push` will re-upload it). See the Recipe Lifecycle section in automations.md.
> 2. Delete the corresponding cookbook config: `DELETE /api/v2/platformModel/cookbook/<orgId>ObjectBuilderMixin/configs/object/$custom_documentTemplate`.
> 3. Republish: `PATCH /api/v2/platformModel/cookbook/<orgId>ObjectBuilderMixin/publish`.
> 4. Re-pull: `bloom recipe:pull --yes`.
>
> After this, both `documentTemplate.documentType` and `document.documentType` extend dynamically based on registered customTypes.

The same gotcha applies to the `document` object's enum. Removing `$custom_documentTemplate` appears to unlock both — there is no separate `$custom_document` mixin to delete in observed orgs.

---

## Other platform gaps to know about

A grab bag of related dead-ends discovered while probing this area, marked so readers don't waste time re-discovering them:

- **`getRecordDocuments` doesn't surface API-created system docs.** `POST /actions/document/getRecordDocuments/1.0 {recordId: <parentId>}` returns `[]` even when valid `isSystemGenerated` docs exist with `systemDocParentId: <parentId>`. The lookup uses a different field (likely `recordId`, not `systemDocParentId`) or filters to platform-internally-created docs only. Consequence: the Documents embedded board on the parent record's detail view shows empty even when your docs exist and render correctly.
- **`document.recordId` / `document.objectKey` are silently dropped on writes.** Same root cause — the public write path does not expose parent-linkage fields. Only `isSystemGenerated` + `systemDocParentId` propagate.
- **No render route for arbitrary (objectKey, documentType) pairs.** Probed exhaustively; all 404:
  - `/documents/<customObj>/<recordId>/<customSlug>/pdf` for slug variants `bill_of_lading__c`, `bill_of_lading`, `bol`, etc.
  - `/documents/<templateId>/<recordId>/pdf`
  - `/documents/render?templateId=…&recordId=…&objectKey=…`
  Only the `/documents/<doc-id>/pdf` path (per-document-record) works for custom objects.
- **No public `render` action.** `POST /actions/documentTemplate/render/1.0`, `generatePdf`, `generate`, `preview`, `POST /actions/document/generateForRecord`, `createForRecord`, `createDocument` — all 404. The only documentTemplate action exposed is `setOrgDefault`.
- **DocumentsEmbeddedBoard widget is TS-only.** A widget config with `component: "DocumentsEmbeddedBoard"` and serialized JS actions referencing platform enums (`WidgetActionType`, `RRIcon`, `ButtonType`) does not survive JSON recipe push and can leave the cookbook in a broken 500 state. Use `config-ConfigurableEmbeddedBoard` with `runtimeConfig: {boardKey: "document"}` instead — the platform auto-resolves parent context from URL.
- **Closed base enum values to know.** `ace`, `aci`, `legBillOfLading`, `billOfLading`, `shipmentBillOfLading`, `commercialInvoice`, `complianceWSIB`, `complianceDriver`, `complianceAuto`, `complianceCargo`, `labels`, `invoice`, `offloadManifest`, `preloadManifest`, `paps`, `rateConfirmation`, `ucc128`, `load`. Custom keys (`<snake_case>__c`) work for the documentTemplate / document write path but won't appear in some UI dropdowns.

---

## End-to-end checklist

The five-step recipe from "I have a custom object" to "PDFs auto-generate with my template":

1. **Register the custom doc type.**
   `POST /api/v2/platformModel/documents/customType` with `{label, objectKey, isCustomizable: true, isAutoCreated: true}`. Verify it lands in `<orgId>ObjectBuilderMixin/configs/documentType/<key>`.
2. **Create the documentTemplate with HTML in `templateData`.**
   `POST /api/v2/platformModel/objects {objectKey: "documentTemplate", json: {name, documentType, rootObjectKey}}`, then `PUT` the same record with `{templateData: "<HTML>", isCodeMode: true}`. **GET it back** and verify `rootObjectKey` was not coerced; PUT-override if it was.
3. **Mark it default.**
   `POST /api/v2/platformModel/actions/documentTemplate/setOrgDefault/1.0` with `{recordId, boardId}`. Confirm `isDefault: true` on the template.
4. **Build a Rocket Engine workflow.**
   Trigger on parent record `onCreate`, guard with `$isEmpty` against the parent's documents connection, then `workflowUtils/createRecord/1.0` with the verified `document` shape:
   ```json
   {
     "objectKey": "document",
     "json": {
       "documentType": "<key>",
       "isSystemGenerated": true,
       "systemDocParentId": "<parent record uuid>"
     }
   }
   ```
5. **Test.**
   Create a parent record. Inspect `/api/v2/platformModel/events?recordId=<parent>` to confirm the workflow ran. GET the resulting document and hit its `externalUrl` (`/api/v2/platformModel/documents/<doc-id>/pdf`) — it should return `application/pdf` rendered against the parent.

---

## Cross-references

- `api-platformmodel.md` — base CRUD shapes for `/objects`, `/objects/{id}`, search.
- `workflows-rocket-engine.md` — Rocket Engine workflow authoring (the auto-create workaround).
- `debugging-automations.md` — `/events` API for inspecting workflow runs.
- `automations.md` — `bloom recipe:push` re-uploads local mixin files; delete locally too.
