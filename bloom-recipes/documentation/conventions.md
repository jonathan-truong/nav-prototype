# Conventions & Cross-Cutting Patterns

## Naming Conventions

| Pattern | Usage | Example |
|---------|-------|---------|
| `__c` suffix | Custom (user-defined) keys | `bike__c`, `colour__c`, `chairBoard__c` |
| `$custom_` prefix | Mixin configs | `$custom_partner`, `$custom_admin` |
| `$` prefix on fields | System-managed fields | `$createdAt`, `$id`, `$version` |
| No prefix/suffix | Built-in platform keys | `order`, `org`, `audit` |

### Key Naming by Config Type

| Config Type | Key Pattern | Example |
|-------------|-------------|---------|
| Object | `camelCase__c` | `bike__c`, `gasstation__c` |
| Board | `camelCaseBoard__c` | `chairBoard__c`, `gasstationBoard__c` |
| Detail View | `camelCaseDetailView__c` | `chairDetailView__c` |
| Group | `$custom_kebab-case` | `$custom_admin`, `$custom_super-admin` |
| Widget | `camelCase__c` | `userDuration__c` |
| Widget Sequence | `$widgetsequence` or key | `$widgetsequence` |

### File Naming

| Config Type | File Pattern | Example |
|-------------|-------------|---------|
| Object | `<name>.json` or `$custom_<name>.json` | `chair.json`, `$custom_partner.json` |
| Board | `<name>-board.json` or `$custom_<name>.json` | `chair-board.json`, `$custom_partner.json` |
| Detail View | `<name>-detail-view.json` or `$custom_<name>.json` | `chair-detail-view.json` |
| Group | `$custom_<name>.json` | `$custom_admin.json` |
| Widget | `<name>.json` | `user-duration.json` |
| Widget Sequence | `<name>.json` or `$<name>.json` | `$widgetsequence.json` |

## configKey Identifiers

Every JSON object in the config system has a `configKey` that identifies its type.

### Top-Level Config Types

| configKey | Description |
|-----------|-------------|
| `"object"` | Data model definition |
| `"board"` | Table/grid UI view |
| `"detailView"` | Single-record detail screen |
| `"group"` | Permission group |
| `"widget"` | Interactive UI component |
| `"widgetSequence"` | Multi-step widget flow |

### Nested Config Types

| configKey | Parent | Description |
|-----------|--------|-------------|
| `"field"` | object.fields[] | Field definition |
| `"event"` | object.events[] | Event definition |
| `"method"` | object.methods[] | Method/automation definition |
| `"column"` | board.columns[] | Board column |
| `"filter"` | board.filters[] | Board filter |
| `"detailSection"` | detailView.sections[] | Detail view section |
| `"detailField"` | detailSection.fields[] | Field in a detail section |
| `"widgetRef"` | widgetSequence.widgets[] | Widget reference in a sequence |

### Permission Config Types (in groups)

| configKey | Parent | Description |
|-----------|--------|-------------|
| `"BoardPermission"` | group.boards[] | Board access permission |
| `"ObjectPermission"` | group.objects[] | Object CRUD permission |
| `"ObjectFieldPermission"` | ObjectPermission.fields[] | Field-level permission |
| `"DetailViewPermission"` | group.detailViews[] | Detail view access |
| `"WidgetSequencePermission"` | group.widgetSequence[] | Widget sequence access |
| `"WidgetPermission"` | WidgetSequencePermission.widgets[] | Individual widget access |

## Ordering: The `before` Linked List

Many arrays use a `before` property to define display order as a linked list:

```json
[
  { "key": "first",  "before": "second" },
  { "key": "second", "before": "third" },
  { "key": "third" }
]
```

- The last item in the list either omits `before` or sets `"before": null`.
- Used in: group.boards[], group.objects[], group.detailViews[], ObjectPermission.fields[], detailSection.fields[], object.fields[] (some).

## `$declarator` Pattern

Tracks which object or mixin originally declared a field, event, or method:

```json
{
  "$declarator": {
    "key": "audit",
    "configKey": "object"
  }
}
```

Common declarator keys:
- `"audit"` — system audit fields ($createdAt, $updatedAt, etc.)
- `"identity"` — identity fields ($id, $externalId)
- `"globalMixin"` — global fields (isActive)
- `"canvasMixin"` — rich text canvas field
- `"template"` — template fields (isTemplate, templateName)
- `"shareable"` — sharing fields (sourceId, subscriptionIds)
- `"withAiEvents"` — AI event declarations
- `"withGenericWebhookEvents"` — webhook event declarations
- `"documentCreationGlobalMixin"` — document creation methods
- `"aiRecordMixin"` — AI prompt suggestions
- `"rowStyleMixin"` — row color customization
- The object's own key (e.g., `"gasstation__c"`) — fields/methods defined directly on this object

## `$stub` Pattern

Marks properties that contain runtime-resolved code (TypeScript functions). In the JSON config, these are represented as:

```json
{
  "action": { "$stub": true },
  "payload": { "$stub": true },
  "objectConstructor": { "$stub": true },
  "persistenceMapping": { "$stub": true }
}
```

Stubs appear in: event payloads, method actions, object constructors, persistence mappings, and derived field values.

## Mixin vs Standalone

| Property | Mixin | Standalone |
|----------|-------|------------|
| `isMixin` | `true` | `false` |
| `targetKey` | Present (e.g., `"partner"`, `"admin"`) | Absent |
| Key prefix | `$custom_` | No prefix |
| Purpose | Extends/overrides a base config | Self-contained config |

Mixins are composed at runtime by the platform's RecipeLoader. A mixin's `targetKey` references the base config it extends.

## Field Type Formats

Field types can appear in two formats, but **the ref format is required for new objects**. The simple string format exists in older configs but does not work correctly at runtime (e.g., record creation will fail silently).

**Reference object** (required for new objects):
```json
{
  "type": {
    "key": "text",
    "ref": true,
    "configKey": "type"
  }
}
```

**Simple string** (legacy only — do not use for new objects):
```json
{ "type": "text" }
```

> **Important:** Always use the ref format when creating or updating objects. The simple string format may pass validation but will cause runtime failures such as the board create button not working.

## Audit Fields

Every top-level config includes:

```json
{
  "createdAt": "2026-02-23T20:13:25.075Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-23T20:13:25.000Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

- Timestamps: ISO 8601 UTC format
- User IDs: UUID format

## Filename Rules

- The `__c` suffix is **dropped** when converting object/board/widget keys to disk filenames. `customer__c` becomes `customer.json`, NOT `customer__c.json`.
- The `$` prefix is **dropped** too — `$custom_admin` writes to `custom-admin.json`, NOT `$custom_admin.json`.
- If both filename variants exist in the same folder (e.g. `$custom_admin.json` left behind by an older bloom version + `custom-admin.json` written by current bloom), `recipe:push` sees them as two configs with the same key and validation fails with `SDLC-4002: Attempting to store duplicate config`. Bloom's reconcile pass (PR #44) catches these for `configRows/` only — `groups/`, `objects/`, `boards/`, `detailViews/`, `widgets/`, `widgetSequences/` are NOT reconciled, so legacy files in those directories must be cleaned up by hand. Surface symptom: SDLC-4002 from `recipe:validate`. Fix: `ls` the directory, identify the legacy filename (typically the one with a `$` or `__c` not in kebab-case form), and delete it.
- **Filenames must not contain `/` characters.** `recipe:pull` derives filenames from the config's labelId; if a labelId contains a slash (e.g., `"won/lost"`) the CLI tries to use it as a path separator and crashes with ENOENT, leaving the local recipe partial. Until the CLI sanitizes slashes, avoid them in labelIds.

## CLI Working Directory

Run `bloom` from the **workspace root** (the directory containing `bloom-recipes/`). Running it from inside `bloom-recipes/<orgId>/` or any other subfolder causes the CLI to treat that folder as a separate empty workspace — pulls write to the wrong place, pushes find no configs to send.

## Refactor Pattern: Text-Cache → Ref + LOOKUP

A common platform anti-pattern: storing a denormalized "cached" copy of a related record's field as a plain text field, kept in sync via `copyValue` automation. This drifts.

The right pattern: replace the text cache with an object ref + a LOOKUP-derived field (often reusing the same field key to avoid breaking widget/board references).

**Before:**
```json
{ "key": "customerName__c", "type": "text" }
```
plus an automation that fires on `order.customer__c` change and copies `customer__c.name` to `customerName__c`.

**After:**
```json
{
  "key": "customer__c",
  "type": { "key": "object", "ref": true, "configKey": "type" },
  "typeConfig": { "objectKey": "customer", "reverse": "orders__c" }
},
{
  "key": "customerName__c",
  "type": { "key": "text", "ref": true, "configKey": "type" },
  "typeConfig": {
    "derivationType": "LOOKUP",
    "lookupExpression": "LOOKUP({customer__c.name})"
  }
}
```

The LOOKUP-derived field updates reactively. Drop the sync automation. Move the LOOKUP from `widget.fields[]` to the detail view's `config-RecordDataTable` (widgets can't render derived fields without 400s).
