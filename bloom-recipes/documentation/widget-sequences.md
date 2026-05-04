# Widget Sequence Configuration

Widget sequences define ordered flows of widgets that are presented together, typically as multi-step forms or workflows.

## File Location

`bloom-recipes/<orgId>/widgetSequences/<name>.json`

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"widgetSequence"` | Yes | Always `"widgetSequence"` |
| `key` | string | Yes | Unique identifier (e.g., `$widgetsequence`) |
| `label` | string | Yes | Display name shown to users |
| `objectKey` | string | Yes | Object this sequence operates on |
| `widgets` | WidgetRef[] | Yes | Ordered array of widget references |
| `isCustom` | boolean | No | `true` for user-defined sequences |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Widget Reference

```json
{
  "key": "userDuration__c",
  "isHidden": false,
  "configKey": "widgetRef",
  "widgetKey": "userDuration__c"
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Reference identifier |
| `widgetKey` | string | Yes | Key of the widget to include |
| `isHidden` | boolean | No | `true` to hide this widget in the sequence |
| `configKey` | `"widgetRef"` | Yes | Always `"widgetRef"` |

## Complete Widget Sequence Example

```json
{
  "configKey": "widgetSequence",
  "key": "$widgetsequence",
  "label": "Complete our survey",
  "widgets": [
    {
      "key": "userDuration__c",
      "isHidden": false,
      "configKey": "widgetRef",
      "widgetKey": "userDuration__c"
    }
  ],
  "isCustom": true,
  "objectKey": "survey__c",
  "createdAt": "2025-06-24T16:11:04.503Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-24T19:37:37.570Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

## Minimal New Widget Sequence Template

```json
{
  "configKey": "widgetSequence",
  "key": "mySequence__c",
  "label": "My Widget Flow",
  "widgets": [
    {
      "key": "firstWidget__c",
      "isHidden": false,
      "configKey": "widgetRef",
      "widgetKey": "firstWidget__c"
    }
  ],
  "isCustom": true,
  "objectKey": "myEntity__c",
  "createdAt": "2026-01-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```

## Relationship to Groups

Widget sequences must be registered in group permissions to be accessible. See [groups.md](groups.md) — Widget Sequence Permission section.
