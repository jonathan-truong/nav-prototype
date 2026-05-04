# Board Configuration

Boards define table/grid views for displaying collections of records.

## File Location

`bloom-recipes/<orgId>/boards/<name>-board.json` (standalone)
`bloom-recipes/<orgId>/boards/$custom_<name>.json` (mixin)

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"board"` | Yes | Always `"board"` |
| `key` | string | Yes | Unique identifier (e.g., `chairBoard__c`) |
| `label` | string | Yes (standalone) | Display name |
| `labelId` | string | No | Label identifier for i18n |
| `icon` | string | No | Emoji icon (e.g., `"🛻"`) |
| `objectKey` | string | Yes (standalone) | Object this board displays (e.g., `chair__c`) |
| `columns` | Column[] | Yes | Array of column definitions |
| `filters` | Filter[] | No | Array of filter definitions |
| `buttons` | Button[] | Yes (standalone) | Array of action buttons. **New standalone boards must include a create button** |
| `isMixin` | boolean | Yes | `false` for standalone, `true` for mixin |
| `isDisabled` | boolean | No | Whether board is hidden |
| `summary` | string | No | Short description |
| `description` | string | No | Longer description |
| `securityFilters` | array | No | Row-level security filters |
| `targetKey` | string | Mixin only | Base board this mixin extends |
| `id` | string | Mixin only | Board identifier |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Column Definition

```json
{
  "id": "fieldKey",
  "key": "fieldKey",
  "path": "fieldKey",
  "label": "Column Label",
  "labelId": "Column Label",
  "configKey": "column",
  "isEditable": true,
  "isEditableInline": true
}
```

### Column Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Column identifier |
| `key` | string | Yes | Field key this column displays |
| `path` | string | Yes | Data path to the field value |
| `label` | string | No | Display label (use `label` or `labelId`) |
| `labelId` | string | No | Label identifier for i18n |
| `configKey` | `"column"` | No | Present in mixin columns |
| `isEditable` | boolean | No | Whether column value can be edited |
| `isEditableInline` | boolean | No | Whether inline editing is enabled |

Note: In simpler boards, columns may omit `configKey`, `isEditable`, and `isEditableInline`.

## Filter Definition

```json
{
  "key": "fieldKey",
  "path": "fieldKey",
  "labelId": "Filter Label",
  "columnKey": "fieldKey",
  "configKey": "filter",
  "isTextSearchIncluded": true
}
```

### Filter Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Filter identifier |
| `path` | string | Yes | Data path to filter on |
| `labelId` | string | Yes | Display label |
| `columnKey` | string | No | Associated column key |
| `configKey` | `"filter"` | No | Present in mixin filters |
| `isTextSearchIncluded` | boolean | No | Whether included in text search |

## Button Definition

```json
{
  "key": "create",
  "type": "primary",
  "action": "async ({ context }): Promise<void> => { ... }",
  "labelId": "Create chairs"
}
```

### Button Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Button identifier (commonly `"create"`) |
| `type` | `"primary"` | Yes | Button **style** (presentation) |
| `kind` | `"single"` \| `"bulk"` | No | Button **category** (action scope). Default `"single"`. Use `"bulk"` for actions that operate on multiple selected rows. |
| `action` | string | No | TypeScript function as string |
| `labelId` | string | Yes | Button label |
| `menuItems` | MenuItem[] | No | Dropdown menu items |

> **Footgun: `kind` vs `type`.** Button categorization is `kind`, not `type`. The TS recipe field `type: ButtonType.Bulk` serializes to JSON as `kind: "bulk"` — direct JSON authoring needs `kind`. Writing `type: "bulk"` silently renders as a primary button (the categorization is dropped at the recipe layer with no validation error).

### Button Action: Create Record

Standard create button action:
```
async ({ context }): Promise<void> => {const resp: { id: string } = await context.createRecord({boardId: context.boardId,objectKey: 'OBJECT_KEY',json: {},});context.navigateToRecord({recordId: resp.id,objectKey: 'OBJECT_KEY',});}
```

Replace `OBJECT_KEY` with the target object key (e.g., `chair__c`).

> **Important:** Every new standalone board must include a create button. Without it, users cannot create records from the board view. See the "Minimal New Board Template" section for a complete example.

> **CRITICAL — Required-fields footgun.** The button's `json` payload must include values for every field on the target object that has `isRequired: true`. An empty `json: {}` passes recipe validation but **fails silently at runtime** when the user clicks the button (createRecord returns a validation error to a context the UI swallows). This is the #1 cause of "the Create button doesn't do anything." Inspect the target object's `fields[]`, identify all required fields (especially refs like `location__c`, `customer__c`), and seed plausible defaults in `json`.

Example with required fields populated:

```
"action": "async ({ context }): Promise<void> => { const resp: { id: string } = await context.createRecord({ boardId: context.boardId, objectKey: 'chair__c', json: { name: 'New Chair', location__c: { id: '<location-uuid>' } } }); context.navigateToRecord({ recordId: resp.id, objectKey: 'chair__c' }); }"
```

> **Embedded boards (`config-ConfigurableEmbeddedBoard`)**: the platform does NOT thread `parentRecordId` through to the embedded board's create button. The action must extract the parent UUID from `window.location.href` (regex on the route segment) and pass it as a ref into the createRecord `json` payload, otherwise the new record is created without a parent and orphans. See `view-builder-guide.md`.

> **Detail-view buttons render but DON'T fire.** JSON-authored buttons at the detail-view level are confirmed broken — they appear in the UI but clicks do nothing. Use a CustomWidget action button instead, or place the action on the board.

For automated diagnosis when a Create button isn't firing, run `/bloom-diagnose-create-button` from Claude Code.

### Menu Item (for dropdown buttons)

Menu items appear in a dropdown attached to the primary button.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `key` | string | Yes | Unique identifier |
| `labelId` | string | Yes | Display label |
| `subtitle` | string | No | Description text below the label |
| `leftIcon` | string | No | Icon identifier (e.g., `"csvImport"`) |
| `onClick` | string | No | TypeScript function as string |
| `widgetKey` | string | No | Opens a widget instead of running onClick |
| `isVisible` | string | No | Visibility function as string |
| `isHidden` | boolean | No | Whether menu item is hidden |

### Menu Item: CSV / Spreadsheet Import

Add a CSV import option as a menu item on the create button:

```json
{
  "key": "openCSVImport",
  "labelId": "CSV / Spreadsheet upload",
  "subtitle": "Upload your data via CSV or spreadsheets in one click",
  "leftIcon": "csvImport",
  "onClick": "async ({ context }): Promise<void> => {context.openDialog('importPlatformDataDialog', {dialogState: {boardId: context.boardId,objectKey: 'OBJECT_KEY',},dialogOptions: {onClose: async () => {await context.refetch();},},});}"
}
```

Replace `OBJECT_KEY` with the board's object key (e.g., `chair__c`).

#### openDialog Parameters for CSV Import

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `dialogState.boardId` | string | Yes | Use `context.boardId` |
| `dialogState.objectKey` | string | Yes | Object key for import target |
| `dialogState.connectionDepth` | number | No | Levels to search for connections (default 1) |
| `dialogState.parentRecordId` | string | No | Parent record ID for nested imports |
| `dialogState.parentObjectKey` | string | No | Parent object key for nested imports |
| `dialogState.parentFieldKey` | string | No | Field key linking imported records to parent |
| `dialogOptions.onClose` | function | No | Callback when dialog closes (typically `context.refetch()`) |

## Standalone Board Example

```json
{
  "configKey": "board",
  "key": "chairBoard__c",
  "icon": "🛻",
  "label": "Chairs Board",
  "buttons": [
    {
      "key": "create",
      "type": "primary",
      "action": "async ({ context }): Promise<void> => {const resp: { id: string } = await context.createRecord({boardId: context.boardId,objectKey: 'chair__c',json: {},});context.navigateToRecord({recordId: resp.id,objectKey: 'chair__c',});}",
      "labelId": "Create chairs",
      "menuItems": [
        {
          "key": "openCSVImport",
          "labelId": "CSV / Spreadsheet upload",
          "subtitle": "Upload your data via CSV or spreadsheets in one click",
          "leftIcon": "csvImport",
          "onClick": "async ({ context }): Promise<void> => {context.openDialog('importPlatformDataDialog', {dialogState: {boardId: context.boardId,objectKey: 'chair__c',},dialogOptions: {onClose: async () => {await context.refetch();},},});}"
        }
      ]
    }
  ],
  "columns": [
    {
      "id": "fullId",
      "key": "fullId",
      "path": "fullId",
      "labelId": "Full ID"
    },
    {
      "id": "name",
      "key": "name",
      "path": "name",
      "labelId": "Name"
    }
  ],
  "filters": [
    {
      "key": "fullId",
      "path": "fullId",
      "labelId": "Full ID",
      "isTextSearchIncluded": true
    },
    {
      "key": "name",
      "path": "name",
      "labelId": "Name",
      "isTextSearchIncluded": true
    }
  ],
  "isMixin": false,
  "labelId": "Chairs",
  "summary": "chairs Board",
  "objectKey": "chair__c",
  "isDisabled": false,
  "description": "chairs Board",
  "securityFilters": [],
  "createdAt": "2026-02-19T18:15:28.341Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-23T20:13:25.000Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

## Mixin Board Example

Mixin boards extend a base board with additional columns/filters. They do not have `objectKey`, `label`, `buttons`, etc.

```json
{
  "configKey": "board",
  "id": "partner",
  "key": "$custom_partner",
  "columns": [
    {
      "key": "contactPersons__c",
      "path": "contactPersons__c",
      "labelId": "Contact Persons",
      "configKey": "column",
      "isEditable": true,
      "isEditableInline": true
    }
  ],
  "filters": [
    {
      "key": "contactPersons__c",
      "path": "contactPersons__c",
      "labelId": "Contact Persons",
      "columnKey": "contactPersons__c",
      "configKey": "filter"
    }
  ],
  "isMixin": true,
  "targetKey": "partner",
  "createdAt": "2026-02-18T13:56:27.179Z",
  "createdBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd",
  "updatedAt": "2026-02-23T20:13:25.000Z",
  "updatedBy": "50e60786-1934-49b2-ad09-c1d6b87cadfd"
}
```

## Multiple Boards for the Same Object

You can create multiple boards that display the same object type. Each board has its own unique `key` and `id` but shares the same `objectKey`. This is useful when you need:

- **Different audiences** — an admin board showing all records vs. a user-scoped board filtered to "my records"
- **Different column sets** — a summary board with key columns vs. a detailed board with all fields
- **Filtered views** — an "Active Orders" board vs. a "Completed Orders" board, each as a top-level nav item

### Rules

1. Each board must have a **unique `key`** and **unique `id`** (e.g., `taskBoard__c` and `taskUserBoard__c`)
2. Both boards use the **same `objectKey`** (e.g., `task__c`)
3. Each board is a **separate JSON file** in `boards/` (e.g., `task-board.json` and `task-user-board.json`)
4. Each board gets its own **navigation item** in the sidebar
5. Each board can link to a **different `detailViewKey`** if needed

### Example: Admin Board (all records)

File: `boards/task-board.json`

```json
{
  "configKey": "board",
  "key": "taskBoard__c",
  "icon": "📋",
  "label": "All Tasks",
  "objectKey": "task__c",
  "isMixin": false,
  "columns": [
    { "id": "fullId", "key": "fullId", "path": "fullId", "labelId": "ID" },
    { "id": "name", "key": "name", "path": "name", "labelId": "Name" },
    { "id": "status", "key": "status", "path": "status", "labelId": "Status" },
    { "id": "assignee", "key": "assignee__c", "path": "assignee__c", "labelId": "Assignee" },
    { "id": "$createdAt", "key": "$createdAt", "path": "$createdAt", "labelId": "Created" }
  ],
  "filters": [
    { "key": "fullId", "path": "fullId", "labelId": "ID", "isTextSearchIncluded": true },
    { "key": "name", "path": "name", "labelId": "Name", "isTextSearchIncluded": true },
    { "key": "status", "path": "status", "labelId": "Status" },
    { "key": "assignee__c", "path": "assignee__c", "labelId": "Assignee" }
  ],
  "buttons": [
    {
      "key": "create",
      "type": "primary",
      "action": "async ({ context }): Promise<void> => {const resp: { id: string } = await context.createRecord({boardId: context.boardId,objectKey: 'task__c',json: {},});context.navigateToRecord({recordId: resp.id,objectKey: 'task__c',});}",
      "labelId": "Create Task"
    }
  ],
  "securityFilters": [],
  "isDisabled": false,
  "createdAt": "2026-03-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-03-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```

### Example: User-Scoped Board (filtered to current user)

File: `boards/task-user-board.json`

This board shows the same `task__c` object but with fewer columns and a security filter that restricts records to those assigned to the current user.

```json
{
  "configKey": "board",
  "key": "taskUserBoard__c",
  "icon": "👤",
  "label": "My Tasks",
  "objectKey": "task__c",
  "isMixin": false,
  "columns": [
    { "id": "fullId", "key": "fullId", "path": "fullId", "labelId": "ID" },
    { "id": "name", "key": "name", "path": "name", "labelId": "Name" },
    { "id": "status", "key": "status", "path": "status", "labelId": "Status" },
    { "id": "dueDate__c", "key": "dueDate__c", "path": "dueDate__c", "labelId": "Due Date" }
  ],
  "filters": [
    { "key": "name", "path": "name", "labelId": "Name", "isTextSearchIncluded": true },
    { "key": "status", "path": "status", "labelId": "Status" }
  ],
  "buttons": [
    {
      "key": "create",
      "type": "primary",
      "action": "async ({ context }): Promise<void> => {const resp: { id: string } = await context.createRecord({boardId: context.boardId,objectKey: 'task__c',json: {},});context.navigateToRecord({recordId: resp.id,objectKey: 'task__c',});}",
      "labelId": "Create Task"
    }
  ],
  "securityFilters": [
    {
      "filters": [
        {
          "filterKey": "assignee__c",
          "path": "assignee__c",
          "operator": "equals",
          "value": "{{currentUserId}}"
        }
      ]
    }
  ],
  "isDisabled": false,
  "createdAt": "2026-03-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-03-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```

### File Naming

When you have multiple boards for the same object, name files to reflect the board's purpose:

| Board Purpose | File Name | Key |
|---------------|-----------|-----|
| All records (admin) | `task-board.json` | `taskBoard__c` |
| User-scoped | `task-user-board.json` | `taskUserBoard__c` |
| Status-filtered | `task-active-board.json` | `taskActiveBoard__c` |
| Embedded in detail view | `task-embedded-board.json` | `taskEmbeddedBoard__c` |

## Minimal New Board Template

```json
{
  "configKey": "board",
  "key": "myEntityBoard__c",
  "icon": "📋",
  "label": "My Entity Board",
  "buttons": [
    {
      "key": "create",
      "type": "primary",
      "action": "async ({ context }): Promise<void> => {const resp: { id: string } = await context.createRecord({boardId: context.boardId,objectKey: 'myEntity__c',json: {},});context.navigateToRecord({recordId: resp.id,objectKey: 'myEntity__c',});}",
      "labelId": "Create My Entity",
      "menuItems": [
        {
          "key": "openCSVImport",
          "labelId": "CSV / Spreadsheet upload",
          "subtitle": "Upload your data via CSV or spreadsheets in one click",
          "leftIcon": "csvImport",
          "onClick": "async ({ context }): Promise<void> => {context.openDialog('importPlatformDataDialog', {dialogState: {boardId: context.boardId,objectKey: 'myEntity__c',},dialogOptions: {onClose: async () => {await context.refetch();},},});}"
        }
      ]
    }
  ],
  "columns": [
    {
      "id": "fullId",
      "key": "fullId",
      "path": "fullId",
      "labelId": "Full ID"
    },
    {
      "id": "name",
      "key": "name",
      "path": "name",
      "labelId": "Name"
    }
  ],
  "filters": [
    {
      "key": "fullId",
      "path": "fullId",
      "labelId": "Full ID",
      "isTextSearchIncluded": true
    },
    {
      "key": "name",
      "path": "name",
      "labelId": "Name",
      "isTextSearchIncluded": true
    }
  ],
  "isMixin": false,
  "labelId": "My Entity",
  "summary": "My Entity Board",
  "objectKey": "myEntity__c",
  "isDisabled": false,
  "description": "My Entity Board",
  "securityFilters": [],
  "createdAt": "2026-01-01T00:00:00.000Z",
  "createdBy": "00000000-0000-0000-0000-000000000000",
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "updatedBy": "00000000-0000-0000-0000-000000000000"
}
```
