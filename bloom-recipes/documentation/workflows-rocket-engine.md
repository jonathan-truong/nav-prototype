# Rocket Engine Workflows

Rocket Engine is the platform's general-purpose, trigger-driven workflow framework. It is the **escape hatch** you reach for whenever per-object `jsActionable` automations cannot do the job — most notably **cross-object record creation** (e.g., creating a `document` record from a custom `__c` object), **document generation** on custom objects, **AI prompt** invocation, **multi-step pipelines with shared state**, and **conditional branching**.

If you have only ever written `jsActionable` automations on an object's `methods[]`, workflows will look familiar at a glance — but they have **two different templating syntaxes** in the same file and a **JSONata gotcha around nested objects** that will silently waste hours if you don't know it. Both are documented below; every example bakes them in.

> Cross-links: see `document-generation.md` for custom-doc-type registration, default-template setup, and the PDF render endpoint. See `automations.md` for the per-object `jsActionable` engine and a comparison of when to use each.

---

## TL;DR — the two footguns

1. **Trigger `inputMapping` uses `{path: "<field>"}` objects, NOT Handlebars.** Writing `{{ $id }}` at trigger level produces the literal string `"{{ $id }}"` in shared state. Node `inputMapping`, by contrast, uses Handlebars `{{ }}` over `input`/`shared`/`output`. Two layers, two syntaxes — get them right.
2. **Nested object literals inside Handlebars MUST use `$merge([...])`.** Handlebars treats `{` as a block opener, so `{"amount": 55, "currencyCode": "USD"}` inline silently breaks parsing with `Mapping failed for fields: ...`. Wrap every nested-object value with `$merge([{a: x}, {b: y}])`.

Read every example below with those two rules in mind.

---

## Where workflows live

```
recipes/<orgId>/
├── workflows/
│   └── <workflowKey>.json    ← one file per workflow
├── installedRecipes.json     ← must include "moduleRocketEngineTier1"
```

Top-level config type: `configKey: "workflow"`. Module: `moduleRocketEngineTier1` (which transitively pulls in `moduleWorkflowBase` for the `workflowUtils/*` action library).

If you don't see `workflows/` after a `recipe:pull`, the org is not on `moduleRocketEngineTier1` yet — add it to `installedRecipes.json` and `recipe:push` before authoring any workflow files.

---

## Top-level shape

```json
{
  "configKey": "workflow",
  "key": "MyWorkflow",
  "json": {
    "key": "MyWorkflow",
    "name": "Human-readable name",
    "description": "What this workflow does and why it exists",
    "configKey": "workflow",
    "queueName": "pipelineAgent",
    "workflowDefinitionVersion": 2,
    "status": "active",

    "triggers": [ /* see Trigger config */ ],
    "inputMapping": { /* top-level — Handlebars, seeds shared state */ },
    "nodes": [ /* see Nodes */ ],
    "edges": [ /* see Edges */ ],

    "runConfig": {
      "resultSummaryLabelExpression": "Created doc {{ shared.createdDocId }}"
    },
    "notifications": { "enabled": false },
    "notes": [],
    "startNodePosition": { "x": 126, "y": 0 },
    "endNodePosition":   { "x": 126, "y": 362 }
  }
}
```

The fields `trigger`, `inputMapping`, `nodes`, `edges`, and `runConfig` are the load-bearing ones — everything else is metadata or UI hints.

---

## The syntax matrix (memorize this)

| Layer | Syntax | Notes |
|---|---|---|
| `trigger.inputMapping` | `{path: "<field>"}` objects | Resolves against the trigger record. **NOT Handlebars.** Plain literals pass through. |
| Top-level `inputMapping` | `{{ input.X }}` Handlebars | Maps the trigger payload into `shared` state. |
| `node.inputMapping` | `{{ ... }}` Handlebars + JSONata | Body is a JS-like expression. References `input`, `shared`, `output`. |
| `node.outputMapping` | `{{ <field> }}` | Plain field names on the action's response. Sets `shared` state. |
| `runConfig.resultSummaryLabelExpression` | `{{ shared.X }}` | Renders the row label in the `workflowRun` board. |

### Trigger `inputMapping` — path objects only

```json
"inputMapping": {
  "recordId":     {"path": "$id"},
  "trailerName":  {"path": "name"},
  "myCustomField":{"path": "myCustomField__c"}
}
```

`{path: "..."}` is recognized by the platform's `buildWorkflowInputParams` (in `queueWorkflow.action.ts`) and resolved via `getValueFromPath` against the trigger record. Plain literals (strings, numbers, booleans) pass through unchanged. Arrays and nested objects recurse.

**Reaching for `{{ $id }}` here is the #1 footgun.** It will not throw — it will store the literal string `"{{ $id }}"` in your shared state, and your downstream nodes will silently use that as the parent record id, producing nonsense data.

### Node `inputMapping` — Handlebars over `input`, `shared`, `output`

```json
"inputMapping": {
  "objectKey": "document",
  "fileName":  "{{ shared.trailerName }} BOL",
  "fieldsToCreate": "{{ [{\"path\": \"documentType\", \"value\": \"ryantest__c\"}, {\"path\": \"systemDocParentId\", \"value\": shared.trailerId}] }}"
}
```

Plain string values (`"document"`) are literals. `{{ ... }}` bodies are full JS-like expressions: array/object literals, arithmetic, string interpolation, and references to:

- `input.X` — workflow input (whatever the top-level `inputMapping` produced)
- `shared.X` — accumulated state across nodes
- `output.X` — the current node's output (only valid in `outputMapping`)

JSONata helpers like `$sum`, `$merge`, `$count`, etc. are available inside the `{{ }}` body.

---

## Nested object literals — `$merge([...])` is mandatory

Handlebars/JSONata treats `{` as a block opener inside `{{ }}`, so an inline nested object as a `value` will silently fail with `Mapping failed for fields: ...` at runtime. Use `$merge([base, additions])` to build the same object via a function call, which the parser handles cleanly.

```jsonata
# WRONG — fails at runtime, even with static values
{"path": "calculatedMileagePay__c", "value": {"amount": 55, "currencyCode": "USD"}}

# RIGHT — money
{"path": "calculatedMileagePay__c", "value": $merge([{"amount": 55}, {"currencyCode": "USD"}])}

# RIGHT — money with computed amount
{"path": "calculatedMileagePay__c",
 "value": $merge([{"amount": $sum(shared.legs.miles) * shared.cpm.amount}, {"currencyCode": "USD"}])}

# RIGHT — dateTime
{"path": "appointmentAt__c",
 "value": $merge([{"dateTimeInLocation": shared.appointment}, {"dateTimeInLocationEnd": null}])}

# RIGHT — connection ref (append to array field)
{"path": "documents__c",
 "value": $merge([{"objectKey": "document", "isAppend": true}, {"recordId": shared.createdDocId}])}
```

This applies anywhere a multi-key object value appears in a workflow expression: `money` (`{amount, currencyCode}`), `dateTime` (`{dateTimeInLocation, dateTimeInLocationEnd}`), connection refs (`{objectKey, recordId, isAppend}`), and any other structured field. **A single nested literal anywhere in your `fieldsToCreate` will break the whole node — you will not get a partial result.**

A working `fieldsToCreate` with a money field, a dateTime field, and a plain text field:

```json
"fieldsToCreate": "{{ [
  {\"path\": \"name\", \"value\": shared.parentName + ' Settlement'},
  {\"path\": \"amountDue__c\", \"value\": $merge([{\"amount\": shared.totalDue}, {\"currencyCode\": \"USD\"}])},
  {\"path\": \"dueAt__c\",     \"value\": $merge([{\"dateTimeInLocation\": shared.dueAt}, {\"dateTimeInLocationEnd\": null}])}
] }}"
```

---

## Trigger config

```json
{
  "key": "Trigger1",
  "type": "automation",
  "configKey": "workflowAutomationTrigger",
  "objectKey": "trailer__c",
  "event": ["create"],
  "condition": {
    "$and": [
      {"status__c": {"$equals": "approved"}},
      {"documents__c": {"$isEmpty": true}}
    ]
  },
  "inputMapping": {
    "recordId": {"path": "$id"},
    "trailerName": {"path": "name"}
  }
}
```

- `event` — array of `["create", "update", "delete"]`. Use `update` + `condition` for state-transition triggers.
- `condition` — Mongo-style filter evaluated against the trigger record. `$and`, `$or`, `$equals`, `$isEmpty`, etc.
- `action` — optional direct trigger action (e.g., `sendNotification`) that runs alongside the workflow. Note: `sendNotification` is valid here at the workflow-trigger action union, even though it is **rejected** by the per-object `jsActionable` action union.

---

## Nodes — the workhorse: `workflowUtils/createRecord/1.0`

Cross-object record creation is the canonical reason to drop into a workflow. Use `workflowUtils/createRecord/1.0` from `moduleWorkflowBase` — it can target **any** object, including `document`, which the per-object `jsActionable` engine cannot.

```json
{
  "key": "createDocNode",
  "actionKey": "workflowUtils/createRecord/1.0",
  "configKey": "workflowBaseNode",
  "type": "base",
  "inputMapping": {
    "objectKey": "document",
    "fieldsToCreate": "{{ [
      {\"path\": \"documentType\",       \"value\": \"ryantest__c\"},
      {\"path\": \"isSystemGenerated\",  \"value\": true},
      {\"path\": \"systemDocParentId\",  \"value\": shared.trailerId},
      {\"path\": \"fileName\",           \"value\": shared.trailerName + ' BOL'}
    ] }}"
  },
  "outputMapping": {
    "createdDocId": "{{ recordId }}"
  }
}
```

Other `workflowUtils` actions (from `moduleWorkflowBase`):

| Action key | Purpose |
|---|---|
| `workflowUtils/createRecord/1.0` | Create a record on any object |
| `workflowUtils/setRecordFields/1.0` | Update fields on an existing record (supports `isAppend` for connection arrays) |
| `workflowUtils/fetchRecordWithPaths/1.0` | Read fields from a record |
| `workflowUtils/lookupRecordByField/1.0` | Find a record by field value |
| `workflowUtils/runAiPrompt/1.0` | Invoke an AI prompt |
| `workflowUtils/queueWorkflow/1.0` | Queue another workflow |
| `workflowUtils/addActivityLogMessage/1.0` | Log to a record's activity feed |

---

## Edges

```json
{ "key": "edgeA", "from": "START",         "to": "createDocNode", "type": "default", "configKey": "workflowDefaultEdge" },
{ "key": "edgeB", "from": "createDocNode", "to": "linkDocNode",   "type": "default", "configKey": "workflowDefaultEdge" },
{ "key": "edgeZ", "from": "linkDocNode",   "to": "END",            "type": "default", "configKey": "workflowDefaultEdge" }
```

Conditional branching:

```json
{
  "key": "edgeBranch",
  "configKey": "workflowConditionalEdge",
  "type": "conditional",
  "from": "decisionNode",
  "to": "<default-target-key>",
  "cases": [
    { "key": "case1", "condition": "<JSONata expression>", "to": "<other-node-key>" }
  ]
}
```

---

## Idempotent doc-generation pattern (one doc per parent)

The most common workflow pattern is **generate one document record per parent record, ever**. Documents render dynamically against the parent — there is no value in re-creating them on every update; multiple records would surface as duplicates in the Documents tab.

The trick: **guard the trigger on the parent's connection array being empty.** Once the workflow runs and appends the new doc id to that array, future updates fail the condition and skip.

```json
"triggers": [{
  "key": "Trigger1",
  "type": "automation",
  "configKey": "workflowAutomationTrigger",
  "objectKey": "trailer__c",
  "event": ["create", "update"],
  "condition": {
    "$and": [
      {"status__c": {"$equals": "approved"}},
      {"documents__c": {"$isEmpty": true}}
    ]
  },
  "inputMapping": {
    "trailerId":   {"path": "$id"},
    "trailerName": {"path": "name"}
  }
}]
```

This requires a typed connection array `documents__c` of object type `document` on the parent — it doubles as the back-link the embedded Documents board uses.

---

## End-to-end example: trailer create → auto-doc-gen → PDF render

Verified working on a custom `trailer__c` object with a custom `ryantest__c` document type. Drop this in as `recipes/<orgId>/workflows/trailer-auto-doc.json`:

```json
{
  "configKey": "workflow",
  "key": "TrailerAutoDoc",
  "json": {
    "key": "TrailerAutoDoc",
    "name": "Trailer auto-doc generator",
    "description": "On trailer__c create (and on update if no doc yet), create a ryantest__c document linked to the trailer and append it to trailer.documents__c.",
    "configKey": "workflow",
    "queueName": "pipelineAgent",
    "workflowDefinitionVersion": 2,
    "status": "active",

    "triggers": [{
      "key": "Trigger1",
      "type": "automation",
      "configKey": "workflowAutomationTrigger",
      "objectKey": "trailer__c",
      "event": ["create", "update"],
      "condition": {
        "$and": [{"documents__c": {"$isEmpty": true}}]
      },
      "inputMapping": {
        "trailerId":   {"path": "$id"},
        "trailerName": {"path": "name"}
      }
    }],

    "inputMapping": {
      "trailerId":   "{{ input.trailerId }}",
      "trailerName": "{{ input.trailerName }}"
    },

    "nodes": [
      {
        "key": "createDoc",
        "actionKey": "workflowUtils/createRecord/1.0",
        "configKey": "workflowBaseNode",
        "type": "base",
        "inputMapping": {
          "objectKey": "document",
          "fieldsToCreate": "{{ [
            {\"path\": \"documentType\",      \"value\": \"ryantest__c\"},
            {\"path\": \"isSystemGenerated\", \"value\": true},
            {\"path\": \"systemDocParentId\", \"value\": shared.trailerId},
            {\"path\": \"fileName\",          \"value\": shared.trailerName + ' BOL'},
            {\"path\": \"name\",              \"value\": shared.trailerName + ' BOL'}
          ] }}"
        },
        "outputMapping": {
          "createdDocId": "{{ recordId }}"
        }
      },
      {
        "key": "linkDoc",
        "actionKey": "workflowUtils/setRecordFields/1.0",
        "configKey": "workflowBaseNode",
        "type": "base",
        "inputMapping": {
          "objectKey": "trailer__c",
          "recordId":  "{{ shared.trailerId }}",
          "fieldsToUpdate": "{{ [
            {\"path\": \"documents__c\",
             \"value\": $merge([{\"objectKey\": \"document\", \"isAppend\": true}, {\"recordId\": shared.createdDocId}])}
          ] }}"
        }
      }
    ],

    "edges": [
      { "key": "e1", "from": "START",     "to": "createDoc", "type": "default", "configKey": "workflowDefaultEdge" },
      { "key": "e2", "from": "createDoc", "to": "linkDoc",   "type": "default", "configKey": "workflowDefaultEdge" },
      { "key": "e3", "from": "linkDoc",   "to": "END",       "type": "default", "configKey": "workflowDefaultEdge" }
    ],

    "runConfig": {
      "resultSummaryLabelExpression": "Created {{ shared.trailerName }} doc {{ shared.createdDocId }}"
    },
    "notifications": { "enabled": false },
    "notes": [],
    "startNodePosition": { "x": 126, "y": 0 },
    "endNodePosition":   { "x": 126, "y": 400 }
  }
}
```

After `bloom recipe:push`, create a trailer. Within seconds the `workflowRun` board (find via `/api/v2/platformModel/boards/admin/all` filtering on `objectKey: workflowRun`) shows a `completed/success` row whose summary label includes the new doc id. `GET /api/v2/platformModel/documents/<doc-id>/pdf` returns the rendered PDF using the org's active default `ryantest__c` template. The doc id is also appended to `trailer.documents__c`, which surfaces it in the Documents embedded board on the trailer detail view.

This unblocks the gap covered in `document-generation.md`: the per-object `jsActionable` createRecord action can't target `document` (`Field "documentType" not found in object "trailer__c"`), but the workflow's createRecord uses `RecordActions.create` internally, which has the full document schema.

---

## Observability

Every workflow fire creates a row in the `workflowRun` board:

- `fullId` — `WF-<n>`
- `workflowKey` — which workflow
- `status` — `pending | running | completed | failed`
- `outcome` — `success | failure`
- `triggerDetails` — `{event, objectKey, triggerKey, triggerType, sourceRecordId}`
- `workflowResultsSummaryLabel` — rendered from `runConfig.resultSummaryLabelExpression`

For trigger-record-side debugging:

```
GET /api/v2/platformModel/events?recordId=<id>&boardId=<board>&objectKey=<obj>
```

Look for `automationResults` entries with `opName` containing `__wf_trigger__<WfKey>__<TriggerKey>` to see whether the trigger fired and what its evaluated input was.

---

## When to choose: workflow vs jsActionable

| Need | Use |
|---|---|
| Same-object `setValue` / `copyValue` | jsActionable (on the object's `methods[]`) |
| Cross-object `createRecord` | **Workflow** (jsActionable's createRecord can't target other objects) |
| Document generation on a base object (`order`, `shipment`) | Platform auto-create automation (no workflow needed) |
| Document generation on a custom `__c` object | **Workflow** — auto-create automation **fails for custom objects** (verified) |
| Sending a notification (toast / email) | jsActionable (`sendNotification` accepted on per-object actions); workflow trigger `action` also works |
| Multi-step pipeline with shared state across steps | **Workflow** |
| AI prompt invocation | **Workflow** (`workflowUtils/runAiPrompt/1.0`) |
| Conditional branching across multiple actions | **Workflow** (`workflowConditionalEdge`) |
| Queueing another automation chain | **Workflow** (`workflowUtils/queueWorkflow/1.0`) |
| Activity-feed logging | **Workflow** (`workflowUtils/addActivityLogMessage/1.0`) |
| Simple compositeAction on a single record | jsActionable |

If your need fits in one row of an object's `methods[]` and operates only on that object plus typed-ref-related children, prefer jsActionable. The moment you need to write to an unrelated object, register a doc, branch, or call AI — switch to a workflow.

---

## Common mistakes (in order of how often they bite)

1. **Handlebars at trigger level.** `{{ $id }}` in `trigger.inputMapping` produces the literal string. Use `{path: "$id"}` instead.
2. **Inline nested objects in a `value`.** `{"amount": 55, "currencyCode": "USD"}` silently fails. Always wrap with `$merge([{a: x}, {b: y}])`.
3. **Forgetting `isAppend: true` on connection-array updates.** Without it, `setRecordFields` overwrites the whole array, blowing away any existing connections.
4. **No idempotency guard on doc-gen workflows.** Without `{documents__c: {$isEmpty: true}}`, every parent update re-creates a duplicate doc.
5. **Module not installed.** No `moduleRocketEngineTier1` in `installedRecipes.json` → workflows pushed but never fire, no error surfaced.
6. **Editing locally and pushing without pulling first.** `recipe:push` is whole-recipe; if a teammate edited the workflow in the Graph editor, your local stale copy will overwrite their changes. Pull-before-push.

---

## Cross-references

- `document-generation.md` — custom-doc-type registration, `documentTemplate` authoring, `setOrgDefault`, the `{isSystemGenerated, systemDocParentId}` write shape that the workflow's createRecord node above relies on.
- `automations.md` — per-object `jsActionable.action` engine, the action union it accepts, and why cross-object createRecord and `document` targeting are blocked there.
- `objects.md` — typed connection arrays (e.g., `documents__c`) and the `reverse` pairing used by the idempotency guard.
