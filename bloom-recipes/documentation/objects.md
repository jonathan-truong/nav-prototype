# Object Configuration

Objects define data models (entities) in the platform. Each object has fields, events, and methods.

## File Location

`bloom-recipes/<orgId>/objects/<name>.json`

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"object"` | Yes | Always `"object"` |
| `key` | string | Yes | Unique identifier. Custom objects use `__c` suffix (e.g., `bike__c`) |
| `label` | string | Yes | Display name |
| `level` | `"primary"` \| `"secondary"` | Yes | Object hierarchy level. **New objects should always use `"primary"`** |
| `fields` | Field[] | Yes | Array of field definitions |
| `events` | Event[] | Yes | Array of event definitions |
| `methods` | Method[] | Yes | Array of method/automation definitions |
| `isMixin` | boolean | Yes | `false` for standalone, `true` for mixin |
| `summary` | string | No | Short description |
| `description` | string | No | Longer description |
| `stability` | `"stable"` | Yes | Stability status. **New objects should always set `"stable"`** |
| `valueType` | `"object"` | No | Always `"object"` |
| `isPrimitive` | `false` | No | Always `false` for objects |
| `isHiddenBoard` | boolean | No | Whether the default board is hidden |
| `objectConstructor` | `{"$stub": true}` | No | Runtime constructor (stub in JSON) |
| `persistenceMapping` | `{"$stub": true}` | No | Database mapping (stub in JSON) |
| `targetKey` | string | Mixin only | Base object this mixin extends |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Field Definition

```json
{
  "key": "fieldName__c",
  "type": { "key": "text", "ref": true, "configKey": "type" },
  "label": "Field Label",
  "isArray": false,
  "isCustom": true,
  "isSystem": false,
  "isHidden": false,
  "isRequired": false,
  "isImportable": true,
  "configKey": "field",
  "typeConfig": {},
  "description": "What this field stores"
}
```

### Field Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Field identifier. Custom: `name__c`. System: `$createdAt` |
| `type` | string \| TypeRef | Yes | Field type (see types below) |
| `label` | string | Yes | Display label |
| `isArray` | boolean | No | `true` for multi-value fields (default `false`) |
| `isCustom` | boolean | No | `true` for user-defined fields |
| `isSystem` | boolean | No | `true` for platform-managed fields |
| `isHidden` | boolean | No | `true` to hide from UI |
| `isRequired` | boolean | No | `true` for mandatory fields |
| `isImportable` | boolean | No | `true` if field can be imported |
| `configKey` | `"field"` | Yes | Always `"field"` |
| `typeConfig` | object | No | Type-specific configuration |
| `description` | string | No | Field description |
| `$declarator` | object | No | Which object/mixin declared this field |
| `shouldDuplicate` | boolean \| null | No | Duplication behavior |
| `copyMethodForDuplication` | string \| null | No | Duplication method |
| `before` | string \| null | No | Next field key in ordering |

## Field Types Reference

| Type | Description | typeConfig |
|------|-------------|------------|
| `text` | Single/multi-line text | `{"isMultiLine": false}` or `{"isMultiLine": true}` |
| `url` | URL/link field | — |
| `number` | Numeric value | `{"isInteger": true, "unitLabel": ""}` |
| `toggle` | Boolean on/off | — |
| `date` | Date picker | — |
| `dateTime` | Local date + time picker | — |
| `utcDate` | UTC timestamp (system) | — |
| `time` | Time-only field (stored as number, minutes since midnight) | — |
| `select` | Single-select dropdown | `{"options": [...]}` |
| `multiSelect` | Multi-select dropdown | `{"options": [...]}` (field also needs `"isArray": true`) |
| `status` | Status field (select with workflow states) | `{"options": [...]}` |
| `fullId` | Auto-generated ID | `{"format": "CH-{1}"}` where prefix varies |
| `money` | Currency amount | `{"currency": "USD"}` |
| `multiCurrencyMoney` | Multi-currency money field | `{"currencies": [...]}` |
| `measurement` | Value + unit (weight, distance, etc.) | `{"unit": "kg", "precision": 2}` |
| `object` | Connection/relationship to another object | `{"objectKey": "targetObject__c"}` |
| `secret` | Encrypted/hidden field | — |
| `any` | Untyped / flexible | — |

Note: `richText` (canvas) fields use the `object` type with a ref: `{"key":"richText","ref":true,"configKey":"object"}`

## API Write Shapes (CRITICAL)

When writing records via the platformModel API (`POST /api/v2/platformModel/objects`), the JSON `json: {...}` payload MUST use the right shape per field type. These are the high-frequency footguns:

| Field type | Correct shape | Wrong (common mistake) | Failure |
|---|---|---|---|
| `text`, `url`, `number`, `toggle` | scalar (string/number/bool) | — | — |
| `date` | `"YYYY-MM-DD"` | — | — |
| `dateTime` | `{dateTimeInLocation: "YYYY-MM-DDTHH:MM:SS", dateTimeInLocationEnd: null}` | `"2026-05-03T08:15:00Z"` | 400 |
| `money` | `{amount: <int cents>, currencyCode: "USD"}` | `{amount, currency}` | 400 |
| `multiCurrencyMoney` | **DO NOT USE** | any | 500, read-only render |
| `measurement` | `{value, unit}` matching typeConfig | abbreviated unit (`"ft"`) | BRL-1010 |
| `object` ref | `{id: "<uuid>"}` | `"<uuid>"` | "Expected object" |
| `select` / `multiSelect` | exact value from `typeConfig.options` | partial / fuzzy | 400 echoing valid options |

**Notes:**
- `multiCurrencyMoney` is broken — renders read-only in detail views and 500s on write. Always use `money` with a fixed `currencyCode`. If you genuinely need multi-currency support, model it as a `money` field plus a sibling `select` for currency.
- `dateTime` rejects the `Z` UTC suffix. The platform expects a structured object with location-aware datetime; bare ISO strings return 400.
- `measurement` requires `unitCategory`, `defaultUnitType`, and `currentUnitType` in typeConfig. Use full words (`"feet"`, not `"ft"`).
- For long-form API reference (envelope, GET semantics, search quirks), see `api-platformmodel.md`.

## Derivation Types: Formula & LOOKUP

Custom fields can be **derived** — values are computed from other fields rather than written directly. Two derivation types are supported:

**`LOOKUP`** — pulls a value from a related record via dot-traversal through an object ref.

```json
{
  "key": "destFacilityAddress__c",
  "type": { "key": "text", "ref": true, "configKey": "type" },
  "isCustom": true,
  "configKey": "field",
  "typeConfig": {
    "derivationType": "LOOKUP",
    "lookupExpression": "LOOKUP({destFacility__c.addressLine1__c})"
  }
}
```

Multi-hop dot-paths work (e.g., `load__c.customer__c.billingAddress__c`). LOOKUP-derived fields update reactively when the source path changes — no sync automation required.

**`formula`** — computed value via expression with aggregation/transformation functions.

Supported functions: `sum, count, average, product, maximum, minimum, concat, dateDiff, convertCurrency`. Money formulas return a value keyed by currency code.

```json
{
  "typeConfig": {
    "derivationType": "formula",
    "formulaExpression": "sum({lineItems__c.amount__c})"
  }
}
```

**Rules**:
- Derived fields are **not writable**. Do not put them in `widget.fields[]` (PUT fails with 400) or in automation `copyValue` `toPath` (silent no-fire).
- Display derived fields in the detail view's `config-RecordDataTable`, never in widgets.
- Set the source components; the derivation cascades.

## Connection Fields (Object Refs)

Fields with `type.key: "object"` model a reference to another object. For paired (1:N or M:N) connections, declare `reverse: "<otherFieldKey>"` on **one** side (the array side is the natural choice) — the platform auto-pairs the inverse on the target.

```json
{
  "key": "lineItems__c",
  "type": { "key": "object", "ref": true, "configKey": "type" },
  "isArray": true,
  "isCustom": true,
  "configKey": "field",
  "typeConfig": {
    "objectKey": "lineItem__c",
    "reverse": "parent__c"
  }
}
```

Multi-hop refs (object → ref → ref → ref) are supported; you can also LOOKUP through 2-hop paths.

## Enum/Select Fields: Strict Validation

`select` and `multiSelect` fields validate writes strictly against `typeConfig.options`. The API returns 400 with **all valid options listed verbatim** — read the error and use one exact value.

```
400 Bad Request: Invalid value "draft-mode" for field status__c.
Valid options: ["draft", "approved", "rejected", "archived"]
```

## Don't Recreate Platform-Shipped Objects

`customer`, `carrier`, `contact`, `location`, `address` (and others) are **platform-shipped objects**. To add custom fields/events/methods, extend them via a mixin file (`$custom_<name>.json` with `isMixin: true` and `targetKey: "<name>"`) — do **NOT** create a parallel `*__c` custom object. Doing so causes BRL-6001 "Unknown object key" validation chaos and creates two sources of truth.

For the full list of platform objects and the right extension pattern, see `platform-objects-and-mixins.md`.

### Select/MultiSelect Options

```json
{
  "typeConfig": {
    "options": [
      { "value": "gas", "labelId": "Gas" },
      { "value": "premium", "labelId": "Premium" },
      { "value": "diesel", "labelId": "Diesel" }
    ]
  }
}
```

Option properties:
- `value` (string, required): Internal value
- `labelId` (string, required): Display label
- `isCustom` (boolean, optional): Whether user-defined

### Type Format

**Always use the reference object format** when creating or updating fields:
```json
{
  "type": {
    "key": "text",
    "ref": true,
    "configKey": "type"
  }
}
```

For connection fields that reference another object, use `"configKey": "object"`:
```json
{
  "type": {
    "key": "targetObject__c",
    "ref": true,
    "configKey": "object"
  }
}
```

Note: `richText` also uses `"configKey": "object"` in its ref form.

> **Warning:** A simple string format (`"type": "text"`) exists in legacy configs but does not work correctly at runtime for new objects. Always use the ref format.

## System Fields

These are automatically included via mixins. You generally do not add them manually.

| Key | Type | Declarator | Description |
|-----|------|------------|-------------|
| `$id` | text | `identity` | Record UUID |
| `$externalId` | text | `identity` | External system ID |
| `$createdAt` | utcDate | `audit` | Creation timestamp |
| `$createdBy` | text | `audit` | Creator user ID |
| `$createdByFullName` | text | `audit` | Creator display name |
| `$updatedAt` | utcDate | `audit` | Last update timestamp |
| `$updatedBy` | text | `audit` | Last updater user ID |
| `$updatedByFullName` | text | `audit` | Last updater display name |
| `$deletedAt` | utcDate | `audit` | Deletion timestamp |
| `$deletedBy` | text | `audit` | Deleter user ID |
| `$ownerId` | text | `audit` | Owner user ID |
| `$version` | number | `audit` | Record version counter |
| `$source` | text | `audit` | Record origin |
| `fullId` | fullId | varies | Human-readable auto-ID |
| `name` | text | varies | Record name |
| `canvas` | richText | `canvasMixin` | Rich text document |
| `isActive` | toggle | `globalMixin` | Active/inactive flag |
| `isTemplate` | toggle | `template` | Template flag (hidden) |
| `templateName` | text | `template` | Template name (hidden) |
| `sourceId` | text | `shareable` | Source record for shared records |
| `subscriptionIds` | text[] | `shareable` | Subscription IDs |
| `rowColorMap` | any | `rowStyleMixin` | Board row colors |
| `llmPromptSuggestions` | text[] | `aiRecordMixin` | AI prompt suggestions (hidden) |

## Event Definition

```json
{
  "key": "withGenericWebhookEvents.record.created",
  "label": "Record Created",
  "typeId": "record.created",
  "payload": { "$stub": true },
  "webhooks": { "enabled": true },
  "activityLog": { "enabled": false },
  "configKey": "event",
  "$declarator": {
    "key": "withGenericWebhookEvents",
    "configKey": "object"
  },
  "description": "Event that is triggered when a record is created"
}
```

### Standard Event Types

| typeId | Description |
|--------|-------------|
| `record.created` | Triggered when a record is created |
| `record.updated` | Triggered when a record changes |
| `record.deleted` | Triggered when a record is deleted |
| `ai.activityLog.prompt` | AI user prompt event |
| `ai.activityLog.completion` | AI completion event |

## Method Definition

Methods are automations triggered by events.

### Basic Method (stub action)

```json
{
  "key": "queueCreatedEvent",
  "event": "create",
  "action": { "$stub": true },
  "labelId": "queueCreatedEvent",
  "configKey": "method",
  "permission": "insecure",
  "$declarator": {
    "key": "withGenericWebhookEvents",
    "configKey": "object"
  }
}
```

### Method with jsActionable (no-code automation)

```json
{
  "key": "Gas_Station_Auto_Set_types",
  "event": "create",
  "label": "Gas Station Auto Set types",
  "action": { "$stub": true },
  "summary": "",
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": {
    "key": "gasstation__c",
    "configKey": "object"
  },
  "jsActionable": {
    "action": {
      "toPath": "fuelTypes__c",
      "actionKey": "setValue",
      "literalValue": ["gas", "premium", "diesel"]
    },
    "condition": {
      "condition": {},
      "description": ""
    }
  }
}
```

### Method Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Unique method identifier |
| `event` | `"create"` \| `"commit"` \| `"delete"` | No | Trigger event (omit for callable methods) |
| `action` | `{"$stub": true}` | Yes | Runtime action code |
| `label` | string | No | Display label |
| `labelId` | string | No | Label identifier |
| `configKey` | `"method"` | Yes | Always `"method"` |
| `permission` | `"insecure"` | No | Permission level |
| `isSystem` | boolean | No | Whether platform-managed |
| `isDisabled` | boolean | No | Whether disabled |
| `$declarator` | object | No | Source mixin/object |
| `jsActionable` | object | No | No-code automation config |

## Creating a New Object

New custom objects should **always** be created with:
- `"level": "primary"` — makes the object a top-level entity with its own board and detail view
- `"stability": "stable"` — marks the object as production-ready

Only use `"secondary"` for objects that are embedded within a primary object and should not have their own board.

## Minimal Object Example

```json
{
  "configKey": "object",
  "key": "myEntity__c",
  "label": "My Entity",
  "level": "primary",
  "stability": "stable",
  "events": [],
  "fields": [
    {
      "key": "fullId",
      "type": { "key": "fullId", "ref": true, "configKey": "type" },
      "label": "Full ID",
      "isArray": false,
      "configKey": "field",
      "isRequired": false,
      "typeConfig": {
        "format": "ME-{1}"
      },
      "isImportable": true
    },
    {
      "key": "name",
      "type": { "key": "text", "ref": true, "configKey": "type" },
      "label": "Name",
      "isArray": false,
      "configKey": "field",
      "isRequired": false,
      "description": "Name of My Entity",
      "isImportable": true
    }
  ],
  "isMixin": false,
  "methods": [],
  "summary": "My Entity",
  "description": "My Entity",
  "createdAt": "2026-01-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```

## Adding a Custom Field to an Existing Object

Append to the object's `fields[]` array:

```json
{
  "key": "myField__c",
  "type": { "key": "text", "ref": true, "configKey": "type" },
  "label": "My Field",
  "isCustom": true,
  "configKey": "field",
  "typeConfig": {
    "isMultiLine": false
  },
  "isImportable": true
}
```

For a select field:

```json
{
  "key": "status__c",
  "type": { "key": "select", "ref": true, "configKey": "type" },
  "label": "Status",
  "isCustom": true,
  "configKey": "field",
  "typeConfig": {
    "options": [
      { "value": "active", "labelId": "Active" },
      { "value": "inactive", "labelId": "Inactive" }
    ]
  },
  "isImportable": true
}
```
