# Group Configuration

Groups define permission sets that control access to boards, objects, fields, detail views, and widgets.

## File Location

`bloom-recipes/<orgId>/groups/$custom_<name>.json`

## Default Permission Groups — Required Role Mapping

> **Do NOT create custom permission groups.** All roles must be mapped to one of the platform's default groups using `targetKey`. Custom group names (e.g., "facility manager", "front desk", "maintenance", "tenant portal") are not valid `targetKey` values and will fail.

The platform provides 11 default roles. When configuring an org, map every user role to the closest default:

| Role | `targetKey` | Best For | Map These Custom Roles Here |
|------|------------|----------|----------------------------|
| Super Admin | `"super-admin"` | Account owners, system administrators. Full access to all objects, settings, integrations, and permissions | System admin, IT admin, platform owner |
| Admin | `"admin"` | Account owners or power users | Office manager, department head, power user |
| Manager | `"manager"` | Senior personnel and accountants | Facility manager, accountant, finance lead |
| Operations | `"operations"` | Dispatchers and freight coordinators | Dispatcher, coordinator, maintenance supervisor, operations lead |
| CSR | `"csr"` | Customer service reps with dedicated accounts | Front desk, support agent, help desk |
| Sales | `"sales"` | Sales team managing customer accounts. Limited financial data access | Sales rep, account executive, BD |
| Driver | `"driver"` | Company drivers and owner operators | Field worker, technician, mobile user |
| Partner | `"partner"` | External partners tracking tenders, providing updates, tracking payables. Limited access | Vendor, contractor, supplier |
| Customer | `"customer"` | Customers accessing orders, tracking status, requesting quotes, invoices. Limited access | Tenant portal, client portal, end user |
| External | `"external"` | External users with limited access | Auditor, consultant, read-only external |
| Guest | `"guest"` | Guest users with very limited access | Public viewer, unauthenticated access |

Group files are named `$custom_<targetKey>.json` (e.g., `$custom_admin.json`, `$custom_super-admin.json`). Each file extends the base role and adds permissions for your custom objects, boards, and detail views.

## Top-Level Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `configKey` | `"group"` | Yes | Always `"group"` |
| `key` | string | Yes | Unique identifier (e.g., `$custom_admin`) |
| `isGroup` | `true` | Yes | Always `true` |
| `isMixin` | `true` | Yes | Always `true` |
| `targetKey` | string | Yes | Role name this group extends (e.g., `"admin"`) |
| `boards` | BoardPermission[] | Yes | Board access permissions |
| `objects` | ObjectPermission[] | Yes | Object CRUD permissions |
| `detailViews` | DetailViewPermission[] | Yes | Detail view access permissions |
| `widgetSequence` | WidgetSequencePermission[] | No | Widget sequence access permissions |
| `createdAt` | string | Yes | ISO 8601 timestamp |
| `createdBy` | string | Yes | UUID of creator |
| `updatedAt` | string | Yes | ISO 8601 timestamp |
| `updatedBy` | string | Yes | UUID of last updater |

## Board Permission

Controls which boards are visible to this group. Ordered via `before` linked list.

```json
{
  "id": "chairBoard__c",
  "key": "chairBoard__c",
  "before": "nextBoardKey",
  "configKey": "BoardPermission",
  "isDisabled": false
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID (usually same as key) |
| `key` | string | Yes | Board key to grant access to |
| `before` | string \| null | No | Next board in display order |
| `configKey` | `"BoardPermission"` | Yes | Always `"BoardPermission"` |
| `isDisabled` | boolean | No | `true` to hide this board from the group |

Note: Built-in boards (e.g., `"orgs"`) may omit `id` and `isDisabled`.

### Board Permission Checklist

When a board is created but users see "board config not found" or the create button doesn't work, it is almost always a **permissions issue**, not a board config issue. Verify all of the following:

1. **BoardPermission** — Group has a `BoardPermission` entry for the board with `isDisabled: false`
2. **ObjectPermission** — Group has an `ObjectPermission` for the object with `isCreateAllowed: true` and `isDeleteAllowed: true` (these are NOT inherited — they must be explicitly set)
3. **ObjectFieldPermission** — All custom fields have `ObjectFieldPermission` entries with `"permissionType": "editor"`
4. **DetailViewPermission** — Group has a `DetailViewPermission` entry for the detail view
5. **Board config** — Board has `isDisabled: false` and a create button with the correct `action` string
6. **Column flags** — Columns that should be editable have `isEditable: true` and optionally `isEditableInline: true`

> **Common mistake:** The board config and button are correct, but the group file is missing the `BoardPermission` entry or the `ObjectPermission` doesn't have `isCreateAllowed: true`. Both the board definition and the group permissions must be configured.

## The 4-Permission Visibility Chain

Every new custom object needs **four** permission types per group, or the corresponding UI surface vanishes. This is the single most common cause of "the board doesn't show up", "columns are read-only", and "widget save returns 403".

| Permission | Gates | Missing → Symptom |
|---|---|---|
| `ObjectPermission` (incl. `fields[]`) | Record CRUD + per-field write | Eye-slash icons on columns; can't create/edit |
| `BoardPermission` | Sidebar visibility | Board doesn't appear in left nav |
| `DetailViewPermission` | Detail page render | 404 "failed to find detail view for role" |
| `WidgetSequencePermission` | Widget render | Widgets don't appear inside detail view |

Add all four to **every** `$custom_<role>.json` group (typical set: `admin`, `super-admin`, `manager`, `operations`, `csr`, `sales`, `driver`, `partner`, `customer`, `external`, `guest`). The Builder UI auto-syncs these; code-pushed changes do not.

> **For an automated audit, run** `bloom recipe:doctor --check perms --object <key>` **or invoke the** `/bloom-check-perms` **skill from Claude Code.**

## Field Permissions: Must Cover Every Group AND System Fields

Two failure modes in this area waste hours:

**1. Missing field perms on a non-admin group** — the field shows an eye-slash icon (looks read-only) on the board for users in that group. Symptom: "the column is read-only for everyone except admins."

Fix: every code-pushed field needs an `ObjectFieldPermission` entry under `objects[].fields[]` for every `$custom_<role>.json`. Use `permissionType: "editor"` for groups expected to write the field; `"viewer"` for read-only; `"none"` to hide.

**2. Missing system fields in field perms** — widget saves return 403 even though the user has perms on every "real" field. Cause: CustomWidget save does a full PUT including system fields (`$createdAt`, `$updatedAt`, `$id`, `$externalId`, etc.); if any of those is missing an `ObjectFieldPermission`, the save aborts.

Fix: include system field entries (`permissionType: "viewer"` is fine for most) on every group.

## DetailViewPermission Requires Both `key` and `detailView`

A common 404 source: a group has a `DetailViewPermission` entry with only `key` and no `detailView`. The platform looks up the detail view by matching `role → group → detailViews[] WHERE entry.detailView IS SET`. Missing the `detailView` field returns 404 "failed to find detail view for role."

Always set BOTH:

```json
{
  "id": "myEntityDetailView__c",
  "key": "myEntityDetailView__c",
  "configKey": "DetailViewPermission",
  "objectKey": "myEntity__c",
  "detailView": "myEntityDetailView__c"
}
```

## WidgetSequencePermission Dual-Entry Pattern

The Builder UI auto-creates **two** `WidgetSequencePermission` entries when you grant a sequence to a group: one keyed by the sequence (`key: "<sequenceKey>"`) and one keyed by the object (`key: "<objectKey>"`). The UI reads the **object-keyed** mirror at runtime; the sequence-keyed entry exists for legacy reasons.

If you only push the sequence-keyed entry (the natural choice when authoring by hand), widgets disappear from the UI. Both entries must exist and stay in sync — same `isDisabled`, same nested `widgets[]`.

Pull from a Builder-UI-configured group and inspect `widgetSequence[]` to see the dual entries; then mirror the pattern in code.

## Read-Only Reference Org

> **Never push, publish, or DELETE against GTI-Canada (UUID `632c4a4e-5843-4626-9535-6e5dfcd2d55f`).** It's a shared read-only reference org containing built-in + curated custom configs. Use `GET` for inspection only.

## Object Permission

Controls CRUD access to an object and its fields.

```json
{
  "id": "gasstation__c",
  "key": "gasstation__c",
  "before": "nextObjectKey",
  "configKey": "ObjectPermission",
  "objectKey": "gasstation__c",
  "isCreateAllowed": true,
  "isDeleteAllowed": true,
  "fields": [
    { ... }
  ]
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID |
| `key` | string | Yes | Object key |
| `before` | string \| null | No | Next object in order |
| `configKey` | `"ObjectPermission"` | Yes | Always `"ObjectPermission"` |
| `objectKey` | string | No | Object key (may duplicate `key`) |
| `isCreateAllowed` | boolean | No | Whether group can create records |
| `isDeleteAllowed` | boolean | No | Whether group can delete records |
| `fields` | ObjectFieldPermission[] | No | Field-level permissions |

Note: Built-in objects (e.g., `"order"`, `"org"`) may omit `id`, `objectKey`, `isCreateAllowed`, `isDeleteAllowed`.

## Object Field Permission

Controls read/write access to individual fields within an object.

```json
{
  "id": "name",
  "key": "name",
  "before": "nextFieldKey",
  "configKey": "ObjectFieldPermission",
  "isDisabled": false,
  "permissionType": "editor"
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID |
| `key` | string | Yes | Field key |
| `before` | string \| null | No | Next field in order |
| `configKey` | `"ObjectFieldPermission"` | Yes | Always `"ObjectFieldPermission"` |
| `isDisabled` | boolean | No | `true` to hide field from group |
| `permissionType` | `"editor"` \| `"viewer"` \| `"none"` | No | Access level |

### Permission Types

| Type | Description |
|------|-------------|
| `"editor"` | Can read and write the field |
| `"viewer"` | Can only read the field |
| `"none"` | No access to the field |

Note: Built-in fields may omit `id`, `isDisabled`, and `permissionType`.

## Detail View Permission

Controls access to detail views. Ordered via `before` linked list.

```json
{
  "id": "gasstationDetailView__c",
  "key": "gasstationDetailView__c",
  "before": "nextDetailViewKey",
  "configKey": "DetailViewPermission",
  "objectKey": "gasstation__c",
  "detailView": "gasstationDetailView__c"
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID |
| `key` | string | Yes | Detail view key |
| `before` | string \| null | No | Next detail view in order |
| `configKey` | `"DetailViewPermission"` | Yes | Always `"DetailViewPermission"` |
| `objectKey` | string | No | Associated object key |
| `detailView` | string | No | Detail view key (usually same as `key`) |

## Widget Sequence Permission

Controls access to widget sequences and individual widgets within them.

```json
{
  "id": "survey__c",
  "key": "survey__c",
  "configKey": "WidgetSequencePermission",
  "objectKey": "survey__c",
  "widgets": [
    {
      "id": "userDuration__c",
      "key": "userDuration__c",
      "configKey": "WidgetPermission",
      "isDisabled": false
    }
  ]
}
```

### Widget Sequence Permission Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID |
| `key` | string | Yes | Widget sequence key |
| `before` | string \| null | No | Next sequence in order |
| `configKey` | `"WidgetSequencePermission"` | Yes | Always `"WidgetSequencePermission"` |
| `objectKey` | string | No | Associated object key |
| `widgets` | WidgetPermission[] | No | Individual widget permissions |

### Widget Permission Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | string | No | Permission entry ID |
| `key` | string | Yes | Widget key |
| `configKey` | `"WidgetPermission"` | Yes | Always `"WidgetPermission"` |
| `isDisabled` | boolean | No | `true` to hide widget from group |

## Full Group Example

See `groups/$custom_admin.json` for a complete example with all permission types.

## Adding a New Object to a Group

To grant a group access to a new object, board, and detail view, add entries to each permission array:

1. Add to `boards[]`:
```json
{
  "id": "myEntityBoard__c",
  "key": "myEntityBoard__c",
  "configKey": "BoardPermission",
  "isDisabled": false
}
```

2. Add to `objects[]`:
```json
{
  "id": "myEntity__c",
  "key": "myEntity__c",
  "fields": [
    {
      "id": "fullId",
      "key": "fullId",
      "configKey": "ObjectFieldPermission",
      "isDisabled": false,
      "permissionType": "editor"
    },
    {
      "id": "name",
      "key": "name",
      "configKey": "ObjectFieldPermission",
      "isDisabled": false,
      "permissionType": "editor"
    }
  ],
  "configKey": "ObjectPermission",
  "objectKey": "myEntity__c",
  "isCreateAllowed": true,
  "isDeleteAllowed": true
}
```

3. Add to `detailViews[]`:
```json
{
  "id": "myEntityDetailView__c",
  "key": "myEntityDetailView__c",
  "configKey": "DetailViewPermission",
  "objectKey": "myEntity__c",
  "detailView": "myEntityDetailView__c"
}
```

4. Update `before` pointers to maintain ordering — the previously-last item's `before` should point to the new entry's key.
