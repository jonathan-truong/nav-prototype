# Detail View Configuration

Detail views define the layout for viewing/editing a single record.

## File Location

`bloom-recipes/<orgId>/detailViews/<name>-detail-view.json` (standalone)
`bloom-recipes/<orgId>/detailViews/$custom_<name>.json` (mixin)

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"detailView"` | Yes | Always `"detailView"` |
| `key` | string | Yes | Unique identifier (e.g., `chairDetailView__c`) |
| `label` | string | No | Display label (can be empty string) |
| `objectKey` | string | Yes (standalone) | Object this view displays (e.g., `chair__c`) |
| `sections` | Section[] | Yes | Array of section definitions |
| `isMixin` | boolean | No | `true` for mixin detail views |
| `targetKey` | string | Mixin only | Base detail view this mixin extends |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Section Definition

```json
{
  "key": "general",
  "label": "Section Title",
  "labelId": "general",
  "configKey": "detailSection",
  "fields": [
    { ... }
  ]
}
```

### Section Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Section identifier (commonly `"general"`) |
| `label` | string | Yes | Display title |
| `labelId` | string | No | Label identifier for i18n |
| `configKey` | `"detailSection"` | Yes | Always `"detailSection"` |
| `fields` | DetailField[] | Yes | Array of field references |

## Detail Field Definition

```json
{
  "key": "fieldKey",
  "path": "fieldKey",
  "configKey": "detailField",
  "isEditable": true,
  "before": "nextFieldKey"
}
```

### Detail Field Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Field key (must match a field on the object) |
| `path` | string | Yes | Data path to the field value |
| `configKey` | `"detailField"` | Yes | Always `"detailField"` |
| `isEditable` | boolean | No | Whether field is editable in this view |
| `before` | string \| null | No | Next field in display order (linked list) |

## Standalone Detail View Example

```json
{
  "configKey": "detailView",
  "key": "gasstationDetailView__c",
  "label": "",
  "sections": [
    {
      "key": "general",
      "label": "Gas Station Details",
      "fields": [
        {
          "key": "location__c",
          "path": "location__c",
          "configKey": "detailField"
        },
        {
          "key": "fuelTypes__c",
          "path": "fuelTypes__c",
          "configKey": "detailField"
        },
        {
          "key": "name",
          "path": "name",
          "configKey": "detailField"
        },
        {
          "key": "fullId",
          "path": "fullId",
          "configKey": "detailField"
        }
      ],
      "labelId": "Gas Station Details",
      "configKey": "detailSection"
    }
  ],
  "objectKey": "gasstation__c",
  "createdAt": "2025-05-23T19:45:44.966Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-24T19:37:37.570Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

## Minimal New Detail View Template

```json
{
  "configKey": "detailView",
  "key": "myEntityDetailView__c",
  "sections": [
    {
      "key": "general",
      "label": "My Entity Details",
      "labelId": "general",
      "configKey": "detailSection",
      "fields": [
        {
          "key": "name",
          "path": "name",
          "configKey": "detailField",
          "isEditable": true
        },
        {
          "key": "fullId",
          "path": "fullId",
          "configKey": "detailField",
          "isEditable": false
        }
      ]
    }
  ],
  "objectKey": "myEntity__c",
  "createdAt": "2026-01-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```

## Platform Widgets Catalog (Embedded into Detail Views)

The legacy `sections` array (above) covers the simple case of a flat field list. For richer layouts (tabbed panels, embedded child boards, sidebar widgets), use the `views` array with platform-provided widgets. See `view-builder-guide.md` for the long-form authoring guide; the shorthand catalog:

| Widget | Purpose | When to use |
|---|---|---|
| `config-ConfigurableEmbeddedBoard` | Embed a child board filtered by a parent connection | Show line items / activity history on a parent record |
| `config-RecordDataTable` | Overview-style key/value table | The canonical "details" panel — including derived/LOOKUP fields |
| `config-WidgetStack` | Sidebar stack of widgets | Right-rail summary widgets |
| `config-RecordCanvas` | Rich-text/notes canvas | Free-form notes attached to a record |

> **`config-ConfigurableEmbeddedBoard` is Zod-locked** to four `runtimeConfig` props: `boardKey`, `label`, `connectedPageFieldPath`, `connectedBoardFieldPath`. Other props (e.g., `hideSearchBar`, `hideFilter`, `hideButtons`) are silently dropped. If you need full control, drop down to the TS-recipe `GenericEmbeddedBoard` widget.

## Detail-View Buttons: BROKEN — Use Widget Actions Instead

> **JSON-authored buttons at the detail-view level RENDER but DON'T FIRE.** Confirmed broken in production. If your design calls for an action button on the detail page, do one of:
> 1. Author the action as a CustomWidget and embed it in the detail view (recommended)
> 2. Move the action to the parent board's button strip
> 3. Drop down to a TS-recipe widget if absolutely necessary

This is a platform gap, not a doc you can fix in a config. Don't waste time debugging "why doesn't my button click do anything?" — it's not you.

## Documents Tab Requires TS-Recipe Today

`DocumentsEmbeddedBoard` is a TypeScript-recipe-only widget. Authoring it in JSON loses the enum bindings (`WidgetActionType`, `RRIcon`, `ButtonType`) and the action serializations break. If you need a documents tab on a detail view today, use the TS-recipe path. The platform team is working on a JSON-native equivalent.

## DetailViewPermission

Every detail view requires a `DetailViewPermission` entry on **every** `$custom_<role>.json`, with **both** `key` AND `detailView` set (matching each other). Missing the `detailView` field returns 404 at runtime. See `groups.md` for the cookbook.

For automated authoring of detail views, use the `/bloom-generate-detail-view` skill from Claude Code.
