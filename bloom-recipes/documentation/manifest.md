# Manifest & Raw Configs

## manifest.json

Tracks all config entries with content hashes for sync operations.

### File Location

`bloom-recipes/<orgId>/manifest.json`

### Structure

```json
{
  "orgId": "07ba4816-29a3-4e5c-a66c-5f54e55a17b4",
  "lastPull": "2026-03-05T02:41:46.728Z",
  "lastPush": null,
  "entries": {
    "<configKey>/<key>": {
      "hash": "b27de2c5163c7eb5",
      "configKey": "<configType>",
      "key": "<configKey>",
      "pulledAt": "2026-03-05T02:41:46.728Z"
    }
  }
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `orgId` | string | Organization UUID |
| `lastPull` | string \| null | ISO 8601 timestamp of last pull from server |
| `lastPush` | string \| null | ISO 8601 timestamp of last push to server |
| `entries` | object | Map of all tracked configs |

### Entry Key Format

`<configType>/<configKey>` — e.g., `object/bike__c`, `board/$custom_partner`, `widget/userDuration__c`

### Entry Properties

| Property | Type | Description |
|----------|------|-------------|
| `hash` | string | Content hash (16-char hex) for change detection |
| `configKey` | string | Config type (`object`, `board`, `detailView`, `group`, `widget`, `widgetSequence`) |
| `key` | string | Config key |
| `pulledAt` | string | ISO 8601 timestamp when this entry was last pulled |

## .raw/configs.json

A single flat JSON array containing all configs from the server.

### File Location

`bloom-recipes/<orgId>/.raw/configs.json`

### Structure

```json
[
  {
    "configKey": "object",
    "key": "bike__c",
    "label": "bike",
    ...
  },
  {
    "configKey": "board",
    "key": "chairBoard__c",
    ...
  },
  ...
]
```

- Contains every config as an element in one array
- Each element has the same structure as its corresponding individual JSON file
- Serves as the raw download before being split into the folder structure
- The folder structure (`objects/`, `boards/`, etc.) is derived from this file by splitting on `configKey`

### Relationship Between Raw and Split Files

| Raw Array Element | Split File Location |
|-------------------|---------------------|
| `configKey: "object"` | `objects/<filename>.json` |
| `configKey: "board"` | `boards/<filename>.json` |
| `configKey: "detailView"` | `detailViews/<filename>.json` |
| `configKey: "group"` | `groups/<filename>.json` |
| `configKey: "widget"` | `widgets/<filename>.json` |
| `configKey: "widgetSequence"` | `widgetSequences/<filename>.json` |
