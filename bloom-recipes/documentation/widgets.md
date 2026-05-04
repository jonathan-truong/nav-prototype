# Widget Configuration

Widgets define interactive UI components that can be embedded in detail views or widget sequences.

## File Location

`bloom-recipes/<orgId>/widgets/<name>.json`

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"widget"` | Yes | Always `"widget"` |
| `key` | string | Yes | Unique identifier (e.g., `userDuration__c`) |
| `objectKey` | string | Yes | Object this widget operates on |
| `component` | `"CustomWidget"` | Yes | React component to render |
| `theme` | `"custom"` | No | Widget theme |
| `version` | number | No | Widget version |
| `isCustom` | boolean | No | `true` for user-defined widgets |
| `fields` | WidgetField[] | No | Fields displayed in the widget |
| `actions` | array | No | Available actions (often empty `[]`) |
| `messages` | Messages | No | i18n translations |
| `condition` | Condition | No | Open/close conditions |
| `dependencies` | array | No | Required dependencies (often empty `[]`) |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Widget Field

```json
{
  "key": "surveyTitle__c",
  "path": "surveyTitle__c",
  "label": "Survey Title"
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Field key from the object |
| `path` | string | Yes | Data path to the field value |
| `label` | string | Yes | Display label |

## Messages (i18n)

```json
{
  "messages": {
    "en": {
      "docs": {
        "description": "How long have they been using product?"
      },
      "title": "User Duration"
    }
  }
}
```

Structure: `messages.<locale>.<section>.<key>`

Common sections:
- `title` — Widget display title
- `docs.description` — Documentation/help text

## Condition

Controls when the widget is shown/available:

```json
{
  "condition": {
    "open": "true",
    "close": "false"
  }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `open` | string | Expression for when widget opens (e.g., `"true"` = always) |
| `close` | string | Expression for when widget closes (e.g., `"false"` = never auto-close) |

## Complete Widget Example

```json
{
  "configKey": "widget",
  "key": "userDuration__c",
  "theme": "custom",
  "fields": [
    {
      "key": "surveyTitle__c",
      "path": "surveyTitle__c",
      "label": "Survey Title"
    }
  ],
  "actions": [],
  "version": 1,
  "isCustom": true,
  "messages": {
    "en": {
      "docs": {
        "description": "How long have they been using product?"
      },
      "title": "User Duration"
    }
  },
  "component": "CustomWidget",
  "condition": {
    "open": "true",
    "close": "false"
  },
  "objectKey": "survey__c",
  "dependencies": [],
  "createdAt": "2025-06-24T16:34:01.952Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-24T19:37:37.570Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

## What You Can't Put In `widget.fields[]`

CustomWidget save does a **full PUT** of every field declared in `fields[]`. That means:

- ❌ **Derived/LOOKUP fields** — fail with 400 Bad Request because derived fields aren't writable. Move them to the detail view's `config-RecordDataTable` instead.
- ❌ **Formula-derived money fields** — same problem; derived value, not directly writable. Set the source components, let the formula cascade.
- ❌ **System fields you don't want users to edit** — including them as widget fields means they get re-PUT on every save, which can clobber audit timestamps. Filter those to the detail view's data table.

✅ **Editable scalars and refs** — text, number, date, dateTime, money components, object refs. These are the natural fit for widget fields.

## Embedded Board Create Button: Parent Ref Pattern

When a `config-ConfigurableEmbeddedBoard` is embedded inside a detail view, the platform does NOT thread `parentRecordId` to the embedded board's create button context. Records created from the embedded board's Create button will be orphaned (no parent ref) unless the action extracts the parent UUID manually.

The pattern: extract the parent UUID from `window.location.href` and pass it as a ref into the `createRecord` payload.

```
"action": "async ({ context }): Promise<void> => { const url = window.location.href; const match = url.match(/\\/records\\/([a-f0-9-]{36})/); const parentId = match ? match[1] : null; if (!parentId) { throw new Error('Could not extract parent UUID from URL'); } const resp: { id: string } = await context.createRecord({ boardId: context.boardId, objectKey: 'lineItem__c', json: { name: 'New Line Item', parent__c: { id: parentId } } }); context.navigateToRecord({ recordId: resp.id, objectKey: 'lineItem__c' }); }"
```

Adjust the regex to match the route segment of the parent record (e.g., `/orders/<uuid>`, `/records/<uuid>`) and substitute the correct ref field key on the child object.

## `config-ConfigurableEmbeddedBoard` Props Are Locked

The platform's `config-ConfigurableEmbeddedBoard` widget exposes only these `runtimeConfig` props:

| Prop | Purpose |
|------|---------|
| `boardKey` | Which board to embed |
| `label` | Tab label |
| `connectedPageFieldPath` | Field on the child object that points to the parent (e.g., `parent__c`) |
| `connectedBoardFieldPath` | Usually `"$id"` — the parent record's id |

Other props you might be tempted to set (`hideSearchBar`, `hideFilter`, `hideButtons`, `hideColumnConfig`, etc.) are silently dropped by the Zod schema. If you need full control, drop down to a TS-recipe `GenericEmbeddedBoard` widget — that's the escape hatch.

## Condition Shapes

The `condition.open` field accepts three shapes; recipe push validation accepts all three but **runtime evaluation has only been verified for the string-boolean and JS-expression forms**.

```json
{ "condition": { "open": "true", "close": "false" } }
```

```json
{ "condition": { "open": "status__c === 'approved'", "close": "false" } }
```

```json
{ "condition": { "open": { "status__c": { "$equals": "approved" } }, "close": "false" } }
```

If you author the mongo-style object form, test it against a real record before relying on it — the validator is permissive and the recipe-push success doesn't prove runtime correctness.

## Minimal New Widget Template

```json
{
  "configKey": "widget",
  "key": "myWidget__c",
  "theme": "custom",
  "fields": [],
  "actions": [],
  "version": 1,
  "isCustom": true,
  "messages": {
    "en": {
      "title": "My Widget",
      "docs": {
        "description": "Description of what this widget does"
      }
    }
  },
  "component": "CustomWidget",
  "condition": {
    "open": "true",
    "close": "false"
  },
  "objectKey": "myEntity__c",
  "dependencies": [],
  "createdAt": "2026-01-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```
